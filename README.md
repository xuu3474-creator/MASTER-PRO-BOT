import json
import ssl
import time
import websocket

# --- আপনার তথ্য এখানে দিন ---
API_TOKEN = "YOUR_API_TOKEN_HERE"  # আপনার ব্রোকার অ্যাকাউন্ট থেকে নেওয়া API Token দিন
TRADE_AMOUNT = 10  # প্রতিটি ট্রেডের অ্যামাউন্ট (ডলার)
DURATION = 1  # ট্রেডের সময়সীমা
DURATION_UNIT = "m"  # 'm' মানে মিনিট, 't' মানে টিক (Ticks)
SYMBOL = "R_100"  # Volatility 100 Index (মার্কেট সিম্বল)
# ---------------------------


def on_open(ws):
    print("🚀 সার্ভারের সাথে কানেক্ট হয়েছে। অথেনটিকেশন করা হচ্ছে...")
    # API টোকেন পাঠিয়ে অ্যাকাউন্টে লগইন করা
    auth_request = {"authorize": API_TOKEN}
    ws.send(json.dumps(auth_request))


def on_message(ws, message):
    data = json.loads(message)

    # ১. লগইন বা অথেনটিকেশন সফল হয়েছে কিনা চেক করা
    if data.get("msg_type") == "authorize":
        if "error" in data:
            print(f"❌ অথেনটিকেশন ব্যর্থ! কারণ: {data['error']['message']}")
            ws.close()
        else:
            print(
                f"✅ লগইন সফল! ব্যবহারকারী: {data['authorize']['email']}"
            )
            print("🤖 অটো ট্রেডিং শুরু হচ্ছে...")

            # উদাহরণ হিসেবে একটি 'CALL' (Up) ট্রেড নেওয়া হচ্ছে
            # আপনি এখানে আপনার নিজস্ব লজিক/ইন্ডিকেটর বসাতে পারেন
            execute_trade(ws, contract_type="CALL")

    # ২. ট্রেড বা বাই (Buy) রিকোয়েস্টের রেসপন্স চেক করা
    elif data.get("msg_type") == "buy":
        if "error" in data:
            print(f"❌ ট্রেড প্লেস করতে ব্যর্থ! কারণ: {data['error']['message']}")
        else:
            print(f"💰 ট্রেড সফলভাবে নেওয়া হয়েছে!")
            print(f"🎯 কন্ট্রাক্ট আইডি (ID): {data['buy']['contract_id']}")
            print(f"📈 ব্যালেন্স এখন: {data['buy']['balance_after']}")

            # ট্রেড নেওয়ার পর বট বন্ধ করে দেওয়া (লুপ এড়াতে)
            ws.close()


def on_error(ws, error):
    print(f"⚠️ ত্রুটি (Error) দেখা দিয়েছে: {error}")


def on_close(ws, close_status_code, close_msg):
    print("🔌 বট বন্ধ করা হয়েছে।")


def execute_trade(ws, contract_type="CALL"):
    """অটোমেটিক ট্রেড প্লেস করার ফাংশন"""
    print(
        f"⏳ {SYMBOL} মার্কেটে {TRADE_AMOUNT}$ এর একটি {contract_type} ট্রেড পাঠানো হচ্ছে..."
    )

    trade_request = {
        "buy": 1,
        "price": TRADE_AMOUNT,
        "parameters": {
            "amount": TRADE_AMOUNT,
            "basis": "stake",
            "contract_type": contract_type,  # 'CALL' অথবা 'PUT'
            "currency": "USD",
            "duration": DURATION,
            "duration_unit": DURATION_UNIT,
            "symbol": SYMBOL,
        },
    }
    ws.send(json.dumps(trade_request))


if __name__ == "__main__":
    # Deriv ব্রোকারের অফিসিয়াল WebSocket URL
    socket_url = "wss://ws.binaryws.com/websockets/v3?app_id=1089"

    # WebSocket কানেকশন তৈরি করা
    ws = websocket.WebSocketApp(
        socket_url,
        on_open=on_open,
        on_message=on_message,
        on_error=on_error,
        on_close=on_close,
    )

    # সিকিউর কানেকশনের জন্য SSL ব্যবহার করে রান করা
    ws.run_forever(sslopt={"cert_reqs": ssl.CERT_NONE})
    
