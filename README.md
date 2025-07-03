# How to Parse Pumpfun Trades with gRPC

> Note: gRPC can be a real headache, but you can skip all the hard parts by using Bitquery's Shred Streams or GraphQL streams. Get started [here](https://ide.bitquery.io/Pumpfun-DEX-Trades-stream)

Let’s say you want to use gRPC to parse Pumpfun trades with very low latency. What are the steps?

Take this swap for example, https://solscan.io/tx/5ZnJRfkpUTr9PVG1enaQW2uedGPxCkrgwctXFSXt8ST9fEhAsz5wVugq8RPeHTzdqc5Sh6aoPBKHiLwr1giYuLJC

![](/solscan.png)

Detecting the core swap requires you to parse all instructions including the ‘Buy’ or ‘Sell’ instruction and the accounts associated with it

![](/solscanlog.png)

The important details like `userBaseTokenReserves`, `userQuoteTokenReserves`, `lpFee` are hidden in the inner instructions `anchor Self CPI Log`

```JSON

{
timestamp:"1751453731"
baseAmountIn:"6492631766"
minQuoteAmountOut:"7500186221773"
userBaseTokenReserves:"6492631766"
userQuoteTokenReserves:"8955124337748"
poolBaseTokenReserves:"448077005164"
poolQuoteTokenReserves:"536873556282630"
quoteAmountOut:"7668181116071"
lpFeeBasisPoints:"20"
lpFee:"15336362233"
protocolFeeBasisPoints:"5"
protocolFee:"3834090559"
quoteAmountOutWithoutLpFee:"7652844753838"
userQuoteAmountOut:"7649010663279"
pool:"2TGfGYdq2uWqq11hSqqPyfWhYmXt7k3NTydKkkKDbFo4"
user:"Fe5EDUgxaTD2J5Htn81Zar2WjxSBwGsS6DXF3v1ycdKT"
userBaseTokenAccount:"2YhkWT9M9MM9V546M1EqBpBkp3xenLemuqLoPjeMehuD"
userQuoteTokenAccount:"ci8dqUG5CoVnfqpdjzcweEpr3zCDLa5AwFB5rfyi6B7"wprotocolFeeRecipient:"7VtfL8fvgNfhz17qKRMjzQEXgbdpnHHHQRh54R9jP2RJ"
protocolFeeRecipientTokenAccount:"DiktBTLuzQnoseBpNykZa4aFXTB6VhaGJpn3CrHEBDGd"
coinCreator:"11111111111111111111111111111111"
coinCreatorFeeBasisPoints:"5"
coinCreatorFee:"0"
}

```

Now, if you were to parse all this data via gRPC, here’s a sample code on how to use python gRPC client to subscribe to logs of the pumpAMM program

```python
import websocket
import json
import threading
import time
import config
import sys

WS_URL = config.my_url
PUMP_FUN_PROGRAM_ID = "pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA"

MAX_RETRIES = 5
INITIAL_RETRY_DELAY = 1  # seconds
retry_count = 0

def send_subscribe_request(ws):
    request = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "logsSubscribe",
        "params": [
            {
                "mentions": [PUMP_FUN_PROGRAM_ID]
            }
        ]
    }
    print("Sending subscription request:")
    print(json.dumps(request, indent=2))
    ws.send(json.dumps(request))

def start_ping(ws):
    def ping():
        while True:
            time.sleep(30)
            try:
                ws.send("ping")
                print("Ping sent")
            except:
                break
    threading.Thread(target=ping, daemon=True).start()

def on_open(ws):
    global retry_count
    print("WebSocket connected")
    retry_count = 0  # Reset retry count on successful connection
    send_subscribe_request(ws)
    start_ping(ws)

def on_message(ws, message):
    try:
        message_obj = json.loads(message)

        # Handle subscription confirmation
        if "result" in message_obj and isinstance(message_obj["result"], int):
            print(f"Successfully subscribed with ID: {message_obj['result']}")
            return

        # Handle actual log data
        if message_obj.get("params", {}).get("result"):
            log_data = message_obj["params"]["result"]
            print("Received log data:")
            print(json.dumps(log_data, indent=2))

            signature = log_data.get("signature")
            if signature:
                print(f"Transaction signature: {signature}")

        else:
            print("Received message:")
            print(json.dumps(message_obj, indent=2))

    except json.JSONDecodeError:
        print("Failed to parse message")

def on_error(ws, error):
    print(f"WebSocket error: {error}")

def on_close(ws, close_status_code, close_msg):
    print("WebSocket closed")
    reconnect()

def reconnect():
    global retry_count
    if retry_count >= MAX_RETRIES:
        print("Max retry attempts reached. Exiting.")
        sys.exit(1)

    delay = INITIAL_RETRY_DELAY * (2 ** retry_count)
    print(f"Attempting to reconnect in {delay} seconds... (Attempt {retry_count + 1}/{MAX_RETRIES})")
    time.sleep(delay)
    retry_count += 1
    connect()

def connect():
    ws = websocket.WebSocketApp(
        WS_URL,
        on_open=on_open,
        on_message=on_message,
        on_error=on_error,
        on_close=on_close
    )
    ws.run_forever()

# Start connection
connect()
```

Remember that the free plans on most tools allow you access only to primitive methods like `logsSubscribe`, you need to do so before you can access their enhanced RPC calls.

Now you have to wade through the transaction argument data to identify the swap amounts and understand the trade.

```
{
"context": {
    "slot": 350803560
},
"value": {
    "signature": "3up5a7NifaY3cMwnjjgwFTeffztRzLb4Awg2BhBMVynGEUrUfkpsrL73RNqFQMbFun3aBz3ns5MspGDyPKogdqJX",
    "err": null,
    "logs": [
        "Program ComputeBudget111111111111111111111111111111 invoke [1]",
        "Program ComputeBudget111111111111111111111111111111 success"
        …
    ]
}
}
```

- Swap token mints
- Swap amounts
- Source and destination wallets

This requires processing the `inner_instructions` and possibly decoding base58-encoded instruction data fields, which vary based on the AMM program used (e.g., Raydium, Pump, Orca, etc.).

Every second you spend processing raw transactions eats away at the low-latency advantage you thought gRPC would give you.

## Skip the Low-Level Parsing with Shred Streams

Shreds are basically parts of a block that is not yet complete.

Bitquery's `solana.dextrades` Kafka stream delivers fully parsed, enriched DEX trade data with sub-second latency, no transaction decoding required. Read more [here](https://docs.bitquery.io/docs/streams/real-time-solana-data/#kafka-stream-by-bitquery).

### What This Means in Practice

Unlike raw gRPC transactions, with Bitquery Kafka DEX Trades:

- You know immediately which protocol (e.g., Pump.fun) executed the trade
- You get token mint addresses for base and quote tokens without parsing instructions
- Amounts, fees, and order details are pre-decoded
- You have both Buy and Sell sides clearly separated
- No guesswork or AMM-specific decoding logic required

![](/stream.png)

You can view and test the data correctness immediately by setting up a GraphQL stream that is powered by Kafka. For example, [this stream](https://ide.bitquery.io/Pumpfun-DEX-Trades-stream) sends all Pumpfun trades in real-time. Just [**sign up on Bitquery IDE**](https://ide.bitquery.io/) and use the developer plan to test.

Below is a summary of all the factors that decide which solution you should opt for when building an enterprise-scale application using on-chain data streams.

![](/comparison.png)
