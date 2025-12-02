#!/usr/bin/env python3
# bot.py - Tbillionz Cloud Trading Bot (Binance + MT4 Bridge)
# Requirements: python-telegram-bot==20.3, ccxt, requests, PyYAML

import asyncio
import logging
import json
import time
from typing import Optional
import requests
import ccxt
import yaml
from decimal import Decimal, ROUND_DOWN

from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# -------------------------
# Load config
# -------------------------
with open("config/config.yaml", "r") as f:
    CFG = yaml.safe_load(f)

BOT_TOKEN = CFG.get("telegram_token")
ALLOWED_USERS = CFG.get("allowed_telegram_user_ids", [])
PAPER_MODE = CFG.get("paper_mode", True)

BINANCE_API_KEY = CFG.get("binance_api_key")
BINANCE_API_SECRET = CFG.get("binance_api_secret")
EXCHANGE_ID = CFG.get("exchange", "binance")
MT4_BRIDGE_URL = CFG.get("mt4_bridge_url")  # e.g. http://vps:5000/trade
MT4_BRIDGE_TOKEN = CFG.get("mt4_bridge_token", "")

# Trading defaults
DEFAULT_RISK_PCT = float(CFG.get("default_risk_pct", 1.0))  # percent of capital per trade
DEFAULT_SL_PTS = int(CFG.get("default_sl_pts", 200))       # points (for forex) or ticks
DEFAULT_TP_PTS = int(CFG.get("default_tp_pts", 400))

# -------------------------
# Logging & state
# -------------------------
logging.basicConfig(format="%(asctime)s %(levelname)s: %(message)s", level=logging.INFO)
log = logging.getLogger("tbillionz_bot")

# In-memory order book for tracking simulated/paper orders
ORDERS = []
BOT_RUNNING = True
RISK_PCT = DEFAULT_RISK_PCT

# -------------------------
# Setup exchange (ccxt)
# -------------------------
exchange = None
if not PAPER_MODE:
    try:
        exchange = getattr(ccxt, EXCHANGE_ID)({
            "apiKey": BINANCE_API_KEY,
            "secret": BINANCE_API_SECRET,
            "enableRateLimit": True,
        })
        exchange.options["createMarketBuyOrderRequiresPrice"] = False
    except Exception as e:
        log.error("Failed to init exchange: %s", e)
        exchange = None

# -------------------------
# Helpers
# -------------------------
def authorized(user_id: int) -> bool:
    if not ALLOWED_USERS:
        return True
    return user_id in ALLOWED_USERS

async def auth_check(update: Update) -> bool:
    uid = update.effective_user.id
    if not authorized(uid):
        try:
            await update.message.reply_text("Unauthorized ❌")
        except: pass
        log.warning("Unauthorized access attempt by %s", uid)
        return False
    return True

def quantize_amount(amount: float, step: Optional[float]) -> float:
    if not step:
        return float(round(amount, 8))
    # step like 0.0001 -> decimal quantize
    s = Decimal(str(step))
    a = Decimal(str(amount)).quantize(s, rounding=ROUND_DOWN)
    return float(a)

def symbol_is_crypto(symbol: str) -> bool:
    # naive: if endswith USDT or BTC etc.
    symbol = symbol.replace("/", "").upper()
    return symbol.endswith("USDT") or symbol.endswith("BUSD") or symbol.endswith("BTC")

# -------------------------
# Execution
# -------------------------
def place_ccxt_market_order(symbol: str, side: str, qty: float):
    # wrapper with simple error handling
    try:
        # ccxt symbol format: "BTC/USDT"
        ccxt_symbol = symbol if "/" in symbol else f"{symbol[:-4]}/{symbol[-4:]}" if len(symbol) > 4 else symbol
        if not exchange:
            raise RuntimeError("Exchange not initialized")
        order = exchange.create_market_order(ccxt_symbol, side.lower(), qty)
        return {"ok": True, "order": order}
    except Exception as e:
        log.exception("ccxt order failed")
        return {"ok": False, "error": str(e)}

def send_to_mt4_bridge(symbol: str, side: str, qty: float, sl: Optional[float]=None, tp: Optional[float]=None):
    payload = {
        "token": MT4_BRIDGE_TOKEN,
        "symbol": symbol,
        "side": side,
        "qty": qty,
        "sl": sl,
        "tp": tp,
    }
    try:
        r = requests.post(MT4_BRIDGE_URL, json=payload, timeout=10)
        return {"ok": r.status_code == 200, "status_code": r.status_code, "text": r.text}
    except Exception as e:
        log.exception("MT4 bridge error")
        return {"ok": False, "error": str(e)}

# Paper order store
def add_paper_order(symbol, side, qty, price=None, sl=None, tp=None):
    order = {
        "id": int(time.time()*1000),
        "symbol": symbol,
        "side": side,
        "qty": qty,
        "price": price,
        "sl": sl,
        "tp": tp,
        "ts": time.time()
    }
    ORDERS.append(order)
    return order

# -------------------------
# Strategy (simple EMA+RSI example)
# -------------------------
import numpy as np
def simple_strategy_decision(ohlcv):
    # ohlcv: list of [ts, o, h, l, c, v]
    closes = np.array([c for _,o,h,l,c,v in ohlcv])
    if len(closes) < 50:
        return None
    ema_fast = np.mean(closes[-9:])
    ema_slow = np.mean(closes[-21:])
    # naive cross
    if ema_fast > ema_slow and (closes[-1] > closes[-2]):
        return "BUY"
    if ema_fast < ema_slow and (closes[-1] < closes[-2]):
        return "SELL"
    return None

# -------------------------
# Telegram Handlers
# -------------------------
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    await update.message.reply_text(f"Tbillionz Bot online. Mode: {'PAPER' if PAPER_MODE else 'LIVE'}")

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    txt = (
        "/status - bot status\n"
        "/trade SYMBOL SIDE QTY [sl] [tp] - execute trade\n"
        "/buy SYMBOL QTY\n"
        "/sell SYMBOL QTY\n"
        "/balance - show exchange/paper balance\n"
        "/orders - list paper/open orders\n"
        "/close_all - close all paper orders (or call bridge to close all on MT4)\n"
        "/paper_on /paper_off - toggle paper mode in config (restart needed)\n"
        "/set_risk PCT - set risk percent per trade\n        "
    )
    await update.message.reply_text(txt)

async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    await update.message.reply_text(f"Running: {BOT_RUNNING}\nPaper mode: {PAPER_MODE}\nOrders tracked: {len(ORDERS)}\nRisk%: {RISK_PCT}")

async def balance(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    if PAPER_MODE:
        await update.message.reply_text("Balance (paper): 10000 USDT")
        return
    if exchange:
        try:
            bal = exchange.fetch_balance()
            usdt = bal.get('total', {}).get('USDT', 0)
            await update.message.reply_text(f"Balance USDT: {usdt}")
        except Exception as e:
            await update.message.reply_text(f"Error fetching balance: {e}")
    else:
        await update.message.reply_text("Exchange not initialized")

async def orders_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    if not ORDERS:
        await update.message.reply_text("No paper orders.")
        return
    text = "\n".join([f"{o['id']} {o['side']} {o['qty']} {o['symbol']}" for o in ORDERS])
    await update.message.reply_text(text)

async def close_all(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    ORDERS.clear()
    await update.message.reply_text("All paper orders cleared (for live, use exchange/MT4 API).")

async def set_risk(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global RISK_PCT
    if not await auth_check(update): return
    args = context.args
    if not args:
        await update.message.reply_text("Usage: /set_risk PERCENT")
        return
    try:
        pct = float(args[0])
        RISK_PCT = pct
        await update.message.reply_text(f"Risk set to {RISK_PCT}%")
    except:
        await update.message.reply_text("Invalid percent")

async def trade(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    args = context.args
    if len(args) < 3:
        await update.message.reply_text("Usage: /trade SYMBOL SIDE QTY [sl] [tp]\nExample: /trade BTC/USDT BUY 0.001")
        return
    symbol = args[0].upper()
    side = args[1].upper()
    qty = float(args[2])
    sl = float(args[3]) if len(args) >= 4 else None
    tp = float(args[4]) if len(args) >= 5 else None

    # Paper mode
    if PAPER_MODE:
        o = add_paper_order(symbol, side, qty, price=None, sl=sl, tp=tp)
        await update.message.reply_text(f"[PAPER] Simulated {side} {qty} {symbol} (id:{o['id']})")
        return

    # Live: route to crypto or MT4
    if symbol_is_crypto(symbol.replace("/", "")):
        # ccxt expects "BTC/USDT"
        s = symbol if "/" in symbol else symbol.replace("", "")
        res = place_ccxt_market_order(s, side, qty)
        if res.get("ok"):
            await update.message.reply_text(f"✅ LIVE {side} {qty} {symbol} executed. {res.get('order')}")
        else:
            await update.message.reply_text(f"❌ Exchange error: {res.get('error')}")
    else:
        res = send_to_mt4_bridge(symbol, side, qty, sl=sl, tp=tp)
        if res.get("ok"):
            await update.message.reply_text(f"✅ Sent to MT4 bridge: {side} {qty} {symbol}")
        else:
            await update.message.reply_text(f"❌ MT4 bridge error: {res}")

# convenience buy/sell
async def buy_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    args = context.args
    if len(args) < 2:
        await update.message.reply_text("Usage: /buy SYMBOL QTY")
        return
    await trade(update, context)

async def sell_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await auth_check(update): return
    args = context.args
    if len(args) < 2:
        await update.message.reply_text("Usage: /sell SYMBOL QTY")
        return
    await trade(update, context)

# -------------------------
# Entry point
# -------------------------
def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(CommandHandler("status", status))
    app.add_handler(CommandHandler("balance", balance))
    app.add_handler(CommandHandler("orders", orders_cmd))
    app.add_handler(CommandHandler("close_all", close_all))
    app.add_handler(CommandHandler("set_risk", set_risk))
    app.add_handler(CommandHandler("trade", trade))
    app.add_handler(CommandHandler("buy", buy_cmd))
    app.add_handler(CommandHandler("sell", sell_cmd))

    log.info("Tbillionz cloud bot starting...")
    app.run_polling()

if __name__ == "__main__":
    main()