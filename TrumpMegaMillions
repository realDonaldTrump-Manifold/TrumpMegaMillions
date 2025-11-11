import json, time, sys, threading
from typing import Any, Dict
import requests
from websocket import WebSocketApp

API = "https://api.manifold.markets/v0"
WS = "wss://api.manifold.markets/ws"
BUY_MULT = 1.15   # overshoot so we likely own ≥1 YES share

TARGET_USERNAME = None
API_KEY = None

def get_user(username: str) -> Dict[str, Any]:
    r = requests.get(f"{API}/user/{username}", timeout=20)
    r.raise_for_status()
    return r.json()  # has 'id'
    # Docs: GET /v0/user/[username]  [oai_citation:1‡Manifold Docs](https://docs.manifold.markets/api)

def get_prob(market_id: str) -> float:
    r = requests.get(f"{API}/market/{market_id}/prob", timeout=15)
    r.raise_for_status()
    return r.json()["prob"]
    # Docs: GET /v0/market/[marketId]/prob  [oai_citation:2‡Manifold Docs](https://docs.manifold.markets/api)

def place_yes(market_id: str, amount: float) -> Dict[str, Any]:
    payload = {
        "amount": round(amount, 4),
        "contractId": market_id,
        "outcome": "YES"
    }
    r = requests.post(
        f"{API}/bet",
        headers={"Authorization": f"Key {API_KEY}", "Content-Type": "application/json"},
        json=payload,
        timeout=30,
    )
    if r.status_code != 200:
        raise RuntimeError(f"bet error {r.status_code}: {r.text}")
    return r.json()
    # Docs: POST /v0/bet  [oai_citation:3‡Manifold Docs](https://docs.manifold.markets/api)

def sell_yes(market_id: str, shares: float | None) -> Dict[str, Any]:
    payload = {"outcome": "YES"}
    if shares is not None:
        payload["shares"] = float(shares)
    r = requests.post(
        f"{API}/market/{market_id}/sell",
        headers={"Authorization": f"Key {API_KEY}", "Content-Type": "application/json"},
        json=payload,
        timeout=30,
    )
    if r.status_code != 200:
        raise RuntimeError(f"sell error {r.status_code}: {r.text}")
    return r.json()
    # Docs: POST /v0/market/[marketId]/sell with 'outcome' and optional 'shares'  [oai_citation:4‡Manifold Docs](https://docs.manifold.markets/api)

def on_message(ws, message):
    try:
        msg = json.loads(message)
    except Exception:
        return
    if msg.get("type") != "broadcast":
        return
    if msg.get("topic") != "global/new-contract":
        return

    data = msg.get("data", {})
    # LiteMarket fields include: id, creatorId, outcomeType, url, probability, etc.  [oai_citation:5‡Manifold Docs](https://docs.manifold.markets/api)
    creator_id = data.get("creatorId")
    if creator_id != on_message.target_user_id:
        return
    if data.get("outcomeType") != "BINARY":
        return

    market_id = data.get("id")
    url = data.get("url")
    question = data.get("question")

    try:
        prob = get_prob(market_id)  # 0..1
        # Approx mana needed to own ≥1 YES share at current prob
        buy_amount = max(0.06, prob) * BUY_MULT
        buy_res = place_yes(market_id, buy_amount)

        # Try to sell exactly 1 share. If fails, sell all.
        try:
            sell_res = sell_yes(market_id, shares=1.0)
        except Exception as e:
            sell_res = sell_yes(market_id, shares=None)

        print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] {question} ({url})")
        print(f"  prob={prob:.3f}  bought≈1 YES via amount={buy_amount:.3f}  then sold 1 YES; OK.")
    except Exception as e:
        print(f"Error on market {market_id}: {e}")

def on_open(ws):
    # Subscribe to global/new-contract and start ping loop
    sub = {"type": "subscribe", "txid": 1, "topics": ["global/new-contract"]}
    ws.send(json.dumps(sub))
    # Ping every 30s per docs.  [oai_citation:6‡Manifold Docs](https://docs.manifold.markets/api)
    def ping():
        txid = 2
        while True:
            time.sleep(30)
            try:
                ws.send(json.dumps({"type": "ping", "txid": txid}))
                txid += 1
            except Exception:
                break
    threading.Thread(target=ping, daemon=True).start()

def on_error(ws, error):
    print("WebSocket error:", error)

def on_close(ws, code, reason):
    print("WebSocket closed:", code, reason)

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: python watch_user_new_markets_buy_and_sell_one_yes.py target_username YOUR_API_KEY")
        sys.exit(1)
    TARGET_USERNAME = sys.argv[1]
    API_KEY = sys.argv[2]

    user = get_user(TARGET_USERNAME)
    target_user_id = user["id"]
    print(f"Watching new markets by @{TARGET_USERNAME} (userId={target_user_id})")

    ws = WebSocketApp(
        WS,
        on_open=on_open,
        on_message=on_message,
        on_error=on_error,
        on_close=on_close,
    )
    on_message.target_user_id = target_user_id
    ws.run_forever()
