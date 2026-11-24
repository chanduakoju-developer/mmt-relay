"""
Tiny relay: TradingView alert (webhook)  ->  this server  ->  MT5 EA polls /latest

Run locally:   pip install flask gunicorn
               RELAY_SECRET=your-secret python relay_server.py
Production:    gunicorn -w 1 -b 0.0.0.0:$PORT relay_server:app     (keep 1 worker)

TradingView alert settings:
  Webhook URL : https://YOUR-HOST/webhook?key=your-secret
  Message     : JSON like the example in the README, or plain text containing buy/sell.
"""
import json
import os
import re
import time

from flask import Flask, jsonify, request

app = Flask(__name__)
SECRET = os.environ.get("RELAY_SECRET", "change-me")

latest = {"seq": 0}
last_raw = {"text": "", "t": 0.0}


def f(d, key):
    try:
        return float(d.get(key) or 0)
    except (TypeError, ValueError):
        return 0.0


@app.post("/webhook")
def webhook():
    if request.args.get("key") != SECRET:
        return "forbidden", 403

    raw = request.get_data(as_text=True).strip()

    # ignore exact duplicates sent within 10 seconds (TradingView occasionally double-fires)
    if raw == last_raw["text"] and time.time() - last_raw["t"] < 10:
        return "duplicate", 200
    last_raw.update(text=raw, t=time.time())

    try:
        data = json.loads(raw)
        if not isinstance(data, dict):
            data = {}
    except ValueError:
        data = {}

    side = str(data.get("side") or data.get("action") or "").lower()
    if side in ("long",):
        side = "buy"
    if side in ("short",):
        side = "sell"

    if side not in ("buy", "sell"):
        text = raw.lower()
        has_buy = re.search(r"\b(buy|long)\b", text) is not None
        has_sell = re.search(r"\b(sell|short)\b", text) is not None
        if has_buy == has_sell:      # neither, or ambiguous -> ignore
            return "ignored", 200
        side = "buy" if has_buy else "sell"

    global latest
    latest = {
        "seq": int(time.time() * 1000),   # ms timestamp: always increasing, survives restarts
        "side": side,
        "symbol": str(data.get("symbol") or data.get("ticker") or ""),
        "entry": f(data, "entry"),
        "sl": f(data, "sl"),
        "tp1": f(data, "tp1"),
        "tp2": f(data, "tp2"),
        "tp3": f(data, "tp3"),
    }
    return "ok", 200


@app.get("/latest")
def get_latest():
    if request.args.get("key") != SECRET:
        return "forbidden", 403
    return jsonify(latest)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8000)))
