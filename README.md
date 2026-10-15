# mt5-tradingview-bridge

Local bridge that copies new trades from the TradingView Strategy Tester into MetaTrader 5. A Chrome extension reads the trade list, a Python server holds the latest signal, and an Expert Advisor places the order.

The bridge runs entirely on your machine and reads the Strategy Tester trade list. A free TradingView account is enough.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![MetaTrader 5](https://img.shields.io/badge/MetaTrader%205-Expert%20Advisor-green.svg)](https://www.metatrader5.com/)

## Risk

This program can send real orders to your broker. Automated trading can lose the account balance, and with leverage the loss can exceed the deposit. Strategy Tester results are not live results: spread, slippage, requotes, latency, and rejected orders all change the outcome.

Test on a demo account until the behavior is familiar. Confirm that your broker allows algorithmic trading. This is an automation tool. Trading decisions and their results remain yours.

Read [DISCLAIMER.md](DISCLAIMER.md) before use. You are responsible for every order this software sends.

## How it works

```
TradingView Strategy Tester (List of Trades)
        |
        |  Chrome extension reads new rows
        v
POST http://127.0.0.1:8080/signal
        |
        |  Bridge keeps one signal in memory
        v
MetaTrader 5 Expert Advisor polls every 750 ms
        |
        v
Market order on the chart symbol
```

All three parts must use the same shared secret. The extension sends it as `X-Bridge-Token`. The Expert Advisor sends the same header. The bridge rejects `/signal` requests that do not match.

## Requirements

- Windows, with MetaTrader 5 installed
- Python 3.8 or newer
- Google Chrome
- A TradingView account (a free account is enough)
- A broker account on which Expert Advisors are allowed

## Layout

```
.
├── bridge/
│   └── bridge.py            # HTTP server and status window
├── extension/               # Chrome extension (load this folder unpacked)
│   ├── manifest.json
│   ├── content.js
│   ├── popup.html
│   └── popup.js
├── mt5/
│   ├── Trading_Bot.mq5      # Expert Advisor source
│   └── Trading_Bot.ex5      # Older compiled build; recompile the .mq5
├── dist/
│   └── TradingBridge.exe    # Legacy build, from before the shared secret
├── requirements.txt
├── DISCLAIMER.md
└── LICENSE
```

`dist/TradingBridge.exe` and `mt5/Trading_Bot.ex5` were built before the shared-secret check. Run `bridge/bridge.py`, and compile `Trading_Bot.mq5` in MetaEditor, so the token is actually enforced.

A short setup video is available at https://youtu.be/Op9VwIgxM8o. It shows an earlier build that started from `TradingBridge.exe`. Follow the steps below for the current source, including the shared secret.

## Setup

### 1. Create a shared secret

Use one random value of at least 16 characters and no line breaks. The bridge, the extension, and the Expert Advisor must all receive that same value.

```powershell
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 2. Start the bridge

From the repository root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:BRIDGE_SECRET = "paste-the-secret-here"
python bridge/bridge.py
```

In Command Prompt, set the secret with `set BRIDGE_SECRET=paste-the-secret-here` instead of `$env:BRIDGE_SECRET`.

Leave the window open. The server listens on `127.0.0.1:8080` only. If Windows Firewall prompts for access, allow it for private networks. If port 8080 is already taken, stop the other process before starting the bridge.

### 3. Load the extension

1. Open `chrome://extensions`.
2. Turn on Developer mode.
3. Choose Load unpacked and select the `extension` folder.
4. Open the extension popup, paste the same secret, and save it.
5. Open TradingView and the Strategy Tester **List of Trades** tab.
6. Allow local network access for the TradingView site when Chrome asks. The page cannot reach `localhost` without that permission.

Keep that TradingView tab loaded. The extension watches the page; it does not receive alerts in the background.

### 4. Install the Expert Advisor

1. Copy `mt5/Trading_Bot.mq5` to:

   `C:\Users\<you>\AppData\Roaming\MetaQuotes\Terminal\<terminal-id>\MQL5\Experts\`

   The terminal data folder is **File → Open Data Folder** in MetaTrader 5. Compile the file in MetaEditor (F7) and confirm the compile succeeds.
2. In MetaTrader 5, open **Tools → Options → Expert Advisors** and add this URL to the WebRequest allow list:

   `http://127.0.0.1:8080`
3. Attach `Trading_Bot` to the chart of the symbol you want to trade.
4. Set **Bridge token** to the same secret. The advisor will not start if the token is shorter than 16 characters.
5. Enable Algo Trading on the toolbar, and allow algorithmic trading in the advisor dialog.

Orders are placed on the chart where the advisor is attached. The signal carries side, stop, target, and volume only.

## Signal format

The extension reads the text of a new trade row.

| Row text | Signal sent |
| --- | --- |
| `BUY SL=50 TP=100 LOT=0.01` | That full string |
| `BUY` or `SELL` | That side only |
| Text containing `long` or `short` | `BUY` or `SELL` |

Anything else is ignored.

`SL` and `TP` are distances in **points** (`SYMBOL_POINT`). On a 5-digit forex symbol, 10 points equal 1 pip. `LOT` is the volume. When the row omits `SL`, `TP`, or `LOT`, the advisor uses its inputs: lot `0.01`, stop `50` points, take-profit `100` points.

To send those values from Pine, put them in the `strategy.entry` id so they show up in the trade list:

```pine
if buyCondition
    entryId = "BUY SL=" + str.tostring(slPoints) +
              " TP=" + str.tostring(tpPoints) +
              " LOT=" + str.tostring(lotSize)
    strategy.entry(entryId, strategy.long)
```

Use the same pattern with `SELL` and `strategy.short`.

## Execution rules

- The advisor polls every 750 ms and acts only when `status` is `NEW`.
- On startup it marks any leftover signal as processed, so an old row is not traded when the advisor is attached.
- The bridge stores one signal. A new POST is rejected while the current signal is still `NEW`. Use **Clear Old Signal** in the bridge window if a legitimate signal stays stuck.
- If the account already has any open position, the signal is discarded and marked processed. The check is `PositionsTotal()`, so a manual position or another advisor also blocks a new entry.
- Volume is clamped to the symbol minimum and maximum.
- The TradingView tab can refresh the whole list without sending those rows again. An identical signal repeated within one second is also dropped.

## HTTP interface

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| `GET` | `/health` | No | Liveness check used by the popup |
| `GET` | `/signal` | Yes | Current signal, timestamp, and status |
| `POST` | `/signal` | Yes | Store `{ "signal": "..." }` |
| `POST` | `/signal/processed` | Yes | Mark the current signal processed |
| `POST` | `/signal/clear` | Yes | Reset the slot to `NONE` |

Authenticated routes require the header `X-Bridge-Token`. Browser calls to `/signal` are accepted only from `https://www.tradingview.com` and `https://tradingview.com`.

## Troubleshooting

**The extension never sends a trade.** Open the Strategy Tester list of trades, reload the TradingView page, and confirm the popup shows the bridge as online. Existing rows are ignored; only a row added after the watcher starts is sent.

**The popup says the bridge is offline.** Start `python bridge/bridge.py` and leave the window open. Check that nothing else is bound to port 8080.

**MetaTrader reports a WebRequest error.** Add `http://127.0.0.1:8080` under Expert Advisors options, and confirm Algo Trading is enabled. The Experts log prints this when the request is blocked.

**The advisor fails on attach.** `BridgeToken` must match `BRIDGE_SECRET` and be at least 16 characters. Compile `Trading_Bot.mq5` in MetaEditor. The included `.ex5` predates the `BridgeToken` input.

**A trade is rejected or missing.** The bridge accepts one pending signal at a time, and the advisor drops a signal when any position is already open. Check the bridge log and the Experts tab for the reason.

## Security

The server binds to loopback only. Broker credentials stay inside MetaTrader. The shared secret is the only access control on `/signal`, so choose a long random value and do not commit it. The extension stores the token in Chrome local storage on that profile.

## Contributing

Open an issue or a pull request. Keep each change limited to the bridge, the extension, or the Expert Advisor.

## License

Released under the [MIT License](LICENSE). Copyright (c) 2025 Nishant Prakash Garg.

This project is independent. It is not affiliated with TradingView, MetaQuotes, or MetaTrader.

## Contact

Telegram: [t.me/teraus111](https://t.me/teraus111)
