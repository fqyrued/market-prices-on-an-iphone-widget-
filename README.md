# market-prices-on-an-iphone-widget-

# Gold & Currency Widget — Technical Documentation

## Overview

This project provides a home screen iOS widget displaying four live financial values:

- Gram gold price (Turkish Lira)
- Gold ounce spot price (USD)
- USD/TRY exchange rate
- EUR/TRY exchange rate

The system consists of a Node.js bridge server (deployed on a VPS) that aggregates data from three independent APIs, and an iOS widget (built with Scriptable) that consumes the server's output.

## Architecture

```
┌─────────────────┐
│  altinapi.com    │──┐
└─────────────────┘  │
┌─────────────────┐  │     ┌──────────────────┐     ┌─────────────┐     ┌──────────────────┐
│  gold-api.com    │──┼────►│  gramapi.js (VPS) │────►│  /price      │────►│  Scriptable widget│
└─────────────────┘  │     │  Node.js + Express│     │  (JSON)      │     │  (iOS home screen)│
┌─────────────────┐  │     └──────────────────┘     └─────────────┘     └──────────────────┘
│ exchangerate.fun │──┘
└─────────────────┘
```

The server is the only component that communicates with external APIs. The widget has no knowledge of the underlying data sources — it polls a single endpoint and renders the result.

## Data Sources

| Field | Source | Update frequency | Authentication | Rate limit |
|---|---|---|---|---|
| `gold_gram_try` | altinapi.com | Weekdays, 08:00–16:00 (Europe/Istanbul) only | API key required | 1000 req/month (free tier) |
| `gold_ons_usd` | gold-api.com | Continuous, 24/5 | None | None (free tier) |
| `usd_try` | exchangerate.fun | Hourly (source-side) | None | None |
| `eur_try` | exchangerate.fun (derived) | Hourly (source-side) | None | None |

`eur_try` is not returned directly by the source. It is calculated as:

```
EUR/TRY = USD/TRY ÷ USD/EUR
```

### Rationale for restricted gold-gram window

altinapi.com's free tier is capped at 1000 requests/month. Restricting fetches to an 8-hour weekday window (rather than running continuously) allows a refresh interval of approximately every 10–11 minutes within that window while staying under the monthly cap. Outside the window, `gold_gram_try` holds its last known value until the window reopens.

```
1000 requests ÷ ~22 trading days/month ≈ 45 requests/day
8 hours × 60 min ÷ 45 ≈ 1 request every ~10–11 minutes
```

## Server Component

### Requirements

- Ubuntu 22.04/24.04 (or similar)
- Node.js 20.x
- Outbound HTTPS access (port 443)
- Inbound access on port 3000 (or behind a reverse proxy)

### Installation

```bash
# 1. Set timezone (required — gold-window logic depends on it)
timedatectl set-timezone Europe/Istanbul
timedatectl    # verify: Time zone: Europe/Istanbul

# 2. Install Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# 3. Set up project
mkdir gold-widget && cd gold-widget
npm init -y
npm install express
```

> **Note on RTC/UTC display:** `timedatectl` will show `RTC time` and `Universal time` as 3 hours behind local Istanbul time. This is expected Linux behavior — the hardware clock (RTC) stores UTC by design, and the OS applies the timezone offset at the application layer. It does not indicate misconfiguration.

Create `gramapi.js` with the server code (see [Server Script](#server-script) below), inserting a valid altinapi.com API key.

### Running the server

The server runs as a `systemd` service, which restarts it automatically on crash and on VPS reboot.

Create `/etc/systemd/system/gramapi.service`:

```ini
[Unit]
Description=Gold & Currency Widget bridge server
After=network.target

[Service]
Type=simple
WorkingDirectory=/projects/ipwidget
ExecStart=/usr/bin/node gramapi.js
Restart=always
RestartSec=5
StandardOutput=append:/projects/ipwidget/output.log
StandardError=append:/projects/ipwidget/output.log
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

> Adjust `WorkingDirectory` and the log file paths if the project lives somewhere other than `/projects/ipwidget`. Confirm the real path to `node` with `which node` before saving.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable gramapi.service
sudo systemctl start gramapi.service
```

### Operations reference

| Task | Command |
|---|---|
| Verify process is running | `sudo systemctl status gramapi.service` |
| Tail logs | `journalctl -u gramapi.service -f` (or `tail -f output.log`) |
| Stop server | `sudo systemctl stop gramapi.service` |
| Restart (after edits) | `sudo systemctl restart gramapi.service` |
| Inspect all running processes | `top` |

### Server Script

`gramapi.js` exposes a single endpoint and runs two independent refresh cycles in memory:

```javascript
const express = require("express");
const app = express();

const ALTINAPI_KEY = "hapi_YOUR_KEY_HERE";

let goldPrice = null;
let onsPrice = null;
let usdTry = null;
let eurTry = null;
let lastUpdated = null;

function istanbulNow() {
  return new Date(new Date().toLocaleString("en-US", { timeZone: "Europe/Istanbul" }));
}

function isGoldWindow() {
  const now = istanbulNow();
  const day = now.getDay();
  const hour = now.getHours();
  return day >= 1 && day <= 5 && hour >= 8 && hour < 16;
}

async function fetchGold() {
  if (!isGoldWindow()) return;
  try {
    const res = await fetch("https://altinapi.com/api/v1/prices", {
      headers: { "X-API-Key": ALTINAPI_KEY }
    });
    const json = await res.json();
    const gold = json.data.find(p => p.symbol === "ALTIN");
    goldPrice = gold?.ask;
  } catch (err) {
    console.log("Gold fetch error:", err.message);
  }
}

async function fetchOns() {
  try {
    const res = await fetch("https://api.gold-api.com/price/XAU");
    const json = await res.json();
    onsPrice = json.price;
  } catch (err) {
    console.log("ONS fetch error:", err.message);
  }
}

async function fetchRates() {
  try {
    const res = await fetch("https://api.exchangerate.fun/latest?base=USD");
    const json = await res.json();
    usdTry = json.rates?.TRY;
    const usdToEur = json.rates?.EUR;
    eurTry = usdToEur ? usdTry / usdToEur : null;
  } catch (err) {
    console.log("Rates fetch error:", err.message);
  }
}

async function refreshAll() {
  await fetchGold();
  await fetchOns();
  await fetchRates();
  lastUpdated = new Date().toISOString();
}

app.get("/price", (req, res) => {
  res.json({
    gold_gram_try: goldPrice,
    gold_ons_usd: onsPrice,
    usd_try: usdTry,
    eur_try: eurTry,
    updated: lastUpdated
  });
});

app.listen(3000, () => console.log("API running on port 3000"));

refreshAll();
setInterval(refreshAll, 11 * 60 * 1000);
```

### API Reference

**`GET /price`**

Response:

```json
{
  "gold_gram_try": 6230.01,
  "gold_ons_usd": 4118.90,
  "usd_try": 32.48,
  "eur_try": 35.20,
  "updated": "2026-06-24T13:00:00.000Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `gold_gram_try` | number \| null | `null` if no successful fetch has occurred yet inside a gold window |
| `gold_ons_usd` | number \| null | |
| `usd_try` | number \| null | |
| `eur_try` | number \| null | Derived field |
| `updated` | string (ISO 8601, UTC) | Timestamp of the most recent refresh cycle, regardless of which individual fields changed |

No authentication is implemented on this endpoint. It is intended for single-device personal use over a private network path (direct IP). Do not expose it publicly without adding authentication.

## iOS Widget Component

### Requirements

- [Scriptable](https://apps.apple.com/app/scriptable/id1405459188) (free, App Store)

### Installation

1. Open Scriptable, create a new script.
2. Paste the widget script (`widget.js`).
3. Set `SERVER_URL` to `http://<server-ip>:3000/price`.
4. Run manually once to verify the preview renders correctly.
5. From the home screen, add a **Medium** Scriptable widget and assign it to this script.

### Refresh behavior

Widget refresh scheduling is controlled by iOS, not by this application. iOS typically refreshes widgets every 15–30 minutes depending on device usage patterns, for system-wide battery management reasons. This applies to all iOS widgets regardless of the host app, and cannot be overridden to force sub-minute refresh intervals.

## Design Decisions and Investigation Log

### Why not read Harem Altın's feed directly

The initial design intent was to source gram-gold pricing directly from Harem Altın's live feed, which most Turkish gold-price platforms ultimately derive from. This was not pursued to completion. Findings, in order of investigation:

1. Displayed prices are populated client-side via a Socket.IO connection (`wss://hrmsocketonly.haremaltin.com/socket.io/`), and are absent from the initial HTML payload.
2. A Node.js Socket.IO client (`socket.io-client`) completed the connection handshake successfully but received no `price_changed` events — indicating server-side filtering of non-browser clients rather than a connection-level failure.
3. A headless browser (Playwright + Chromium), configured to load the page exactly as a standard visitor would, also failed to receive live price data, despite rendering the page's static structure correctly. This suggests detection of automated browser sessions independent of standard WebSocket/User-Agent checks.

This pattern (works in a manual browser, fails identically under multiple automation approaches) is consistent with deliberate anti-automation measures rather than a technical defect. The project was redirected to independent third-party APIs (altinapi.com, gold-api.com) that provide equivalent data through documented, sanctioned access methods.

### Why three separate sources instead of one

No single free, keyless, rate-unlimited source was identified that covers all four required values. The combination used represents the best available trade-off between cost, reliability, and update frequency for each individual field, rather than a single optimal provider.

## Known Limitations

| Limitation | Detail |
|---|---|
| Gold-gram refresh window | Limited to weekdays 08:00–16:00 (Istanbul) to remain within altinapi.com's free-tier request cap. |
| USD/EUR update frequency | Hourly at the source; not a tick-level live feed. |
| Transport security | Server is plain HTTP. Acceptable for single-device personal use; not suitable for exposure beyond that. |
| Widget refresh control | Bounded entirely by iOS's own scheduling; not configurable by this application. |
| Endpoint authentication | None implemented. |

## Potential Improvements

- Restrict SSH to key-based authentication (password authentication is currently enabled).
- Add TLS termination (e.g. via Caddy or nginx) if the endpoint is ever exposed beyond a single trusted device.
- Move to a paid altinapi.com tier if continuous (non-windowed) gold-gram updates are required.
- Add basic endpoint authentication (e.g. a shared token) before any wider exposure.
