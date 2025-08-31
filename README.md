#!/usr/bin/env python3
"""
Crypto Price Telegram Bot
Features:
- /start : Welcome message
- /help  : Show commands
- /price <coin>
- /top   : Top 5 coins
- /info <coin>
- /convert <coin> <currency>
"""

import json, time, traceback
from urllib import request, parse, error

# 🔑 Your Telegram Bot API Token
TOKEN = "Your Token"
API_URL = f"https://api.telegram.org/bot{TOKEN}/"
CG_API = "https://api.coingecko.com/api/v3"
FIAT = "usd"

# --- Basic HTTP Wrappers ---
def http_get(url, params=None):
    if params: url += "?" + parse.urlencode(params)
    with request.urlopen(url) as r:
        return json.load(r)

def http_post(url, data=None):
    data = parse.urlencode(data or {}).encode()
    req = request.Request(url, data=data)
    with request.urlopen(req) as r:
        try: return json.load(r)
        except: return {}

# --- Telegram Helpers ---
def get_updates(offset=None):
    return http_get(API_URL + "getUpdates", {"timeout": 30, "offset": offset})

def send_message(chat_id, text):
    http_post(API_URL + "sendMessage", {"chat_id": chat_id, "text": text})

# --- CoinGecko API Wrappers ---
def cg_price(coin, vs="usd"):
    return http_get(CG_API + "/simple/price", {"ids": coin, "vs_currencies": vs})

def cg_top(vs="usd"):
    return http_get(CG_API + "/coins/markets", {
        "vs_currency": vs, "order": "market_cap_desc",
        "per_page": 5, "page": 1, "price_change_percentage": "24h"
    })

def cg_info(coin, vs="usd"):
    d = http_get(CG_API + "/coins/markets", {
        "vs_currency": vs, "ids": coin, "price_change_percentage": "24h"
    })
    return d[0] if d else None

# --- Command Handlers ---
def handle_command(chat, text):
    parts = text.split()
    cmd, args = parts[0].lower(), parts[1:]

    if cmd in ("/start", "/help"):
        send_message(chat,
            "👋 Welcome to Crypto Bot!\n\n"
            "Commands:\n"
            "/price <coin>\n"
            "/top\n"
            "/info <coin>\n"
            "/convert <coin> <currency>\n"
            "/help"
        )

    elif cmd == "/price" and args:
        try:
            d = cg_price(args[0], FIAT)
            p = d[args[0]][FIAT]
            send_message(chat, f"💰 {args[0]} = {p} {FIAT.upper()}")
        except: send_message(chat, "⚠️ Coin not found")

    elif cmd == "/top":
        try:
            m = cg_top(FIAT)
            msg = "📊 Top 5 Cryptos:\n"
            for x in m:
                msg += f"{x['name']} ({x['symbol'].upper()}): ${x['current_price']} ({x['price_change_percentage_24h']:+.2f}%)\n"
            send_message(chat, msg)
        except: send_message(chat, "⚠️ Error loading top coins")

    elif cmd == "/info" and args:
        try:
            x = cg_info(args[0], FIAT)
            if not x: return send_message(chat, "⚠️ Not found")
            msg = (
                f"ℹ️ {x['name']} ({x['symbol'].upper()})\n"
                f"Price: ${x['current_price']}\n"
                f"24h: {x['price_change_percentage_24h']:+.2f}%\n"
                f"Market Cap: ${x['market_cap']}\n"
                f"Volume: ${x['total_volume']}"
            )
            send_message(chat, msg)
        except: send_message(chat, "⚠️ Error")

    elif cmd == "/convert" and len(args) >= 2:
        try:
            d = cg_price(args[0], args[1])
            p = d[args[0]][args[1]]
            send_message(chat, f"🔄 {args[0]} = {p} {args[1].upper()}")
        except: send_message(chat, "⚠️ Conversion failed")

    else:
        send_message(chat, "❓ Unknown. Try /help")

# --- Main Loop ---
def main():
    print("💹 Crypto Bot is running...")
    offset = None
    while True:
        try:
            updates = get_updates(offset)
            for u in updates.get("result", []):
                offset = u["update_id"] + 1
                msg = u.get("message", {})
                chat = msg.get("chat", {}).get("id")
                txt = msg.get("text", "")
                if chat and txt:
                    try: handle_command(chat, txt)
                    except: traceback.print_exc()
        except KeyboardInterrupt:
            break
        except Exception as e:
            print("Err:", e)
            time.sleep(5)

if __name__ == "__main__":
    main()