<div align="center">

# 📈 PocketStock

**A palm-sized, WiFi-connected stock ticker built from under $10 of parts. Live prices, sparkline charts, and a market clock — on your desk, off your phone.**

<img src="cad/renders/psv1-img1.png" alt="PocketStock render" width="600"/>

*ESP32-C3 · SSD1306 OLED · 3 buttons · Finnhub API · $9.45 BOM · 100% open source*

</div>

---

## What it does

PocketStock is a tiny desktop gadget that pulls **live stock quotes** from the [Finnhub API](https://finnhub.io) over WiFi and displays them on a crisp 0.96" OLED. Scroll through your watchlist, open a ticker, and get:

- 💵 **Live price**, refreshed every 3 seconds
- 📉 **Change since previous close**
- 📊 **Sparkline chart** of the last 20 price updates
- 🕐 **Real-time clock** synced via NTP
- 🔔 **Market open/closed indicator** (knows NYSE hours, including the 9:30 open)

Navigation is a simple three-button menu system — up, down, select — with **long-press to go back**. No touchscreen, no app, no subscription.

Is it going to replace the military-grade supercomputer in your other pocket? No. But it's a dedicated, glanceable market display that doesn't buzz, doesn't distract, and doesn't try to sell you anything. It just tells you the number. (V1 runs off USB power — true pocket portability is on the roadmap.)

Oh, and there's **Pong** hidden in the settings menu. Because every good gadget needs a game.

## Hardware

The entire build costs **$9.45** in parts:

| Ref | Part | Qty | Cost |
|-----|------|:---:|-----:|
| U1 | ESP32-C3 SuperMini | 1 | $6.00 |
| U2 | SSD1306 OLED display, 128×64 I²C (addr `0x3C`) | 1 | $3.00 |
| SW1–SW3 | 6mm tactile push buttons | 3 | $0.45 |
| | **Total** | | **$9.45** |

Plus a custom PCB (gerbers in [`hardware/gerbers`](hardware/gerbers)) and a 3D-printed enclosure (STLs in [`cad/psv1-stls`](cad/psv1-stls)). Full parts list with supplier links: [`hardware/bill-of-materials-psv1.csv`](hardware/bill-of-materials-psv1.csv)

### Wiring

| Signal | ESP32-C3 pin |
|--------|:---:|
| OLED SDA | GPIO 4 |
| OLED SCL | GPIO 5 |
| Button — Up | GPIO 6 |
| Button — Down | GPIO 7 |
| Button — Select | GPIO 8 |

Buttons are active-low using the ESP32's internal pull-ups — wire each button between its GPIO and GND. No external resistors needed.

## Build it

### 1. Print the enclosure

Print the three STLs in [`cad/psv1-stls`](cad/psv1-stls): `base-v3.stl`, `lid-v3.stl`, and `button-v3.stl` (×3).

### 2. Order the PCB

Upload the files in [`hardware/gerbers`](hardware/gerbers) to your favorite fab (JLCPCB, PCBWay, OSH Park). Schematics live in [`hardware/schematics`](hardware/schematics) if you'd rather breadboard it first.

### 3. Flash the firmware

1. Install the [Arduino IDE](https://www.arduino.cc/en/software) and add ESP32 board support (Boards Manager → search "esp32" → install Espressif's package).
2. Install these libraries via Library Manager:
   - `Adafruit SSD1306`
   - `Adafruit GFX Library`
   - `ArduinoJson`
3. Open [`firmware/PocketStockV1/PocketStockV1.ino`](firmware/PocketStockV1/PocketStockV1.ino) and set your own values:

```cpp
const char* ssid   = "YOUR_WIFI_SSID";   // open networks supported; add a password arg to WiFi.begin() for secured ones
const char* apiKey = "YOUR_FINNHUB_KEY"; // free at https://finnhub.io
```

4. Edit the `stocks[]` array to build your own watchlist — any Finnhub-supported ticker works.
5. Select **ESP32C3 Dev Module** as the board, hit upload, and watch the boot logo roll.

> **Note:** Finnhub's free tier allows 60 calls/minute. PocketStock polls every 3 seconds (20/min), so you're well within limits — but get your *own* free key rather than sharing one.

## Controls

| Action | What it does |
|--------|--------------|
| **Up / Down** | Move through menus and your stock list |
| **Select** (short press) | Open the highlighted item |
| **Select** (hold ~0.7s) | Go back |
| **Up / Down** (in Pong) | Move your paddle 🏓 |

## Gallery

<div align="center">
<img src="cad/renders/psv1-img2.png" alt="PocketStock render — alternate angle" width="45%"/>
<img src="cad/renders/psv1-img3.png" alt="PocketStock render — alternate angle" width="45%"/>
</div>

## Repo map

```
├── cad/         3D-printable enclosure (STLs + renders)
├── docs/        Architecture notes
├── firmware/    ESP32-C3 Arduino sketch
├── hardware/    PCB gerbers, schematics, BOM
├── images/      Photos & renders for this README
└── software/    Desktop-side utilities (WIP)
```

## Roadmap

- [ ] WiFi provisioning (no hardcoded SSID)
- [ ] Store API key + watchlist in flash, editable from the device
- [ ] Percent-change view and daily high/low
- [ ] Crypto ticker support
- [ ] Battery power + deep sleep

## License

See [LICENSE](LICENSE). Fork it, print it, mod it — and if you build one, open an issue with a photo. I'd love to see it.

---

<div align="center">
Built by <a href="https://github.com/the2fye">@the2fye</a> · Powered by too much caffeine and the Finnhub free tier
</div>
