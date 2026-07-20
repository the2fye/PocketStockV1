# PocketStock Architecture

This document describes how PocketStock works internally — the hardware layout, the firmware's structure, and how data flows from the Finnhub API to the pixels on the OLED.

## System overview

PocketStock is a single-microcontroller design. An ESP32-C3 SuperMini does everything: it drives the display, reads the buttons, maintains WiFi, fetches stock quotes, keeps time, and runs the UI (and Pong). There is no companion app or server — the device talks directly to two external services over WiFi:

```
[Buttons] ──► ESP32-C3 ──► [SSD1306 OLED]
                 │
                 ├──► Finnhub REST API  (stock quotes, HTTPS)
                 └──► pool.ntp.org      (time sync, NTP)
```

## Hardware

The ESP32-C3 talks to the SSD1306 OLED (128×64, address `0x3C`) over I²C, with SDA on GPIO 4 and SCL on GPIO 5. The three buttons — Up (GPIO 6), Down (GPIO 7), Select (GPIO 8) — are wired between their pins and ground, using the ESP32's internal pull-up resistors. A pressed button reads LOW; no external components are needed.

Power comes over USB. The ESP32-C3 SuperMini's onboard regulator supplies 3.3 V to the display.

## Firmware structure

The firmware (`firmware/PocketStockV1/PocketStockV1.ino`) is a single-file Arduino sketch organized around a **finite state machine**. A `UIState` enum tracks which screen is active:

```
MAIN_MENU ──► STOCK_LIST ──► STOCK_VIEW
    │
    └──► SETTINGS_MENU ──► SETTINGS_WIFI
                      ├──► SETTINGS_ABOUT
                      └──► GAME_PONG
```

Every iteration of `loop()` does three things: interpret button input in the context of the current state, run any per-state periodic work (quote fetching in `STOCK_VIEW`, physics in `GAME_PONG`), and redraw the screen when something changed. Screens are drawn by dedicated `draw*()` functions; drawing happens on demand rather than every frame, except in Pong.

### Input handling

Up and Down are simple polled reads with a 150 ms delay acting as debounce and auto-repeat rate. Select is more interesting: the firmware timestamps the press and measures duration on release. A release under 700 ms is a **short press** (enter/confirm); holding 700 ms or longer is a **long press** (back). This gives three buttons the navigational power of five — no dedicated back button needed, which keeps the PCB and enclosure simpler.

### Quote fetching

While in `STOCK_VIEW`, the firmware polls Finnhub's `/quote` endpoint for the selected ticker every 3 seconds (non-blocking — it checks `millis()` rather than sleeping). The JSON response is parsed with ArduinoJson into current price (`c`) and change since previous close (`d`). At 20 requests/minute this stays comfortably inside Finnhub's free-tier limit of 60/minute.

### Sparkline

Each fetched price is pushed into a 20-slot **ring buffer**. The sparkline renderer finds the min and max across the buffer, then maps each stored price into a 14×40 px region on the right edge of the stock view, connecting consecutive points with lines. Because scaling is relative to the buffered window, the sparkline always uses the full height of its region — it shows the *shape* of recent movement, not absolute scale. The buffer resets when you switch tickers so one stock's history never bleeds into another's.

### Time and market status

On boot, after WiFi connects, the firmware syncs with `pool.ntp.org` (configured for US Eastern, UTC−5 with DST offset). The stock view header shows the current time plus a market indicator: `isMarketOpen()` checks for weekdays between 9:30 AM and 4:00 PM and the display shows an open or closed glyph accordingly. This is a display hint only — the device still fetches quotes when markets are closed; prices simply don't move.

### Pong

Selecting Pong from settings enters a dedicated game loop: player paddle on the left (Up/Down buttons), a simple AI paddle on the right that tracks the ball's vertical position, score incremented per successful return, high score kept for the session. The game runs at roughly 80 fps (12 ms frame delay) and exits via the same long-press-back gesture as everything else.

## Boot sequence

1. Configure button GPIOs and I²C, initialize the display.
2. Show the boot logo (2 s).
3. Connect to WiFi (blocking, with an on-screen message).
4. Sync time via NTP.
5. Enter `MAIN_MENU`.

## Design constraints and tradeoffs

**Single-core simplicity.** Everything runs in one loop with no RTOS tasks. The tradeoff: the HTTPS fetch briefly blocks input, so button presses during the ~200–500 ms of a quote request are dropped. Acceptable for this UI; a future version could move fetching to a FreeRTOS task.

**No persistence.** WiFi credentials, the API key, and the watchlist are compile-time constants; the Pong high score lives in RAM. Storing these in NVS flash (editable from the device) is the top item on the roadmap.

**Polling over streaming.** Finnhub offers a websocket for real-time trades, but REST polling every 3 s is dramatically simpler, easier on RAM, and indistinguishable on a 128×64 display.
