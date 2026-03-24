# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Does

CrossPoint Calendar is a Cloudflare Worker that generates 480×800 grayscale BMP images for e-ink displays (specifically the Xteink X4 / CrossPoint Reader). It fetches calendar events from Google Calendar and weather data from Open-Meteo (with Visual Crossing as fallback), then renders a combined display image in a high-contrast "utilitarian print" aesthetic.

## Development Commands

All commands run from the `worker/` directory:

```bash
npx wrangler dev          # Run locally
npx wrangler deploy       # Deploy to Cloudflare
```

Local development uses `.dev.vars` for secrets (copy from `.dev.vars.example` if present):
```
GOOGLE_CALENDAR_API_KEY=...
GOOGLE_CALENDAR_ID=...
VISUAL_CROSSING_API_KEY=...  # optional fallback
```

There are no test or lint scripts configured.

## Architecture

The entire worker lives in a single file: `worker/src/index.ts`.

**Request flow:**
1. Cloudflare Worker receives HTTP GET
2. Weather (Open-Meteo → Visual Crossing fallback) and calendar events (Google Calendar → mock data fallback) are fetched in parallel
3. `renderDisplay()` draws everything onto a flat `Uint8Array` pixel buffer
4. `createBMP()` wraps the buffer in an 8-bit grayscale BMP header
5. Response returns `image/bmp`

**Key functions in `index.ts`:**
- `fetchWeather()` — orchestrates weather with 15-min KV cache
- `fetchCalendarEvents()` — fetches from Google Calendar API
- `getMockEvents()` — fallback when no API key is configured
- `renderDisplay()` — main rendering entry point
- `createBMP()` — generates the BMP file bytes
- Drawing primitives: `drawText()`, `drawChar()`, `drawWeatherIcon()`, `fillRect()`, `drawHLine()`, `drawDashedHLine()`

**Configuration (wrangler.toml vars):**
- `DISPLAY_WIDTH` / `DISPLAY_HEIGHT` — defaults to 480×800
- `GOOGLE_CALENDAR_API_KEY`, `GOOGLE_CALENDAR_ID` — optional; falls back to mock events
- `VISUAL_CROSSING_API_KEY` — optional weather fallback

**Hardcoded constants** (change in `index.ts` if needed):
- Location: Basel, CH (47.5596°N, 7.5886°E)
- Timezone: `Europe/Zurich`
- Weather cache TTL: 15 min (errors: 5 min)

**Transit pages** (added feature):
- `fetchTramDepartures()` — fetches Tram 8 from `transport.opendata.ch` (no API key needed)
- `renderTransitPage()` — renders weather + departure list
- Time windows: 08:00–10:30 and 14:00–19:00 Basel time show Tram 8 Laupenring → SBB Basel; all other times show calendar

## Pending Setup (TODO at home)

### 1. Cloudflare KV — fill in wrangler.toml

`worker/wrangler.toml` has placeholder KV namespace IDs. After `npx wrangler login`:

```bash
cd worker
npx wrangler kv namespace create "WEATHER_CACHE"          # → copy id
npx wrangler kv namespace create "WEATHER_CACHE" --preview # → copy preview_id
```

Replace in `worker/wrangler.toml`:
- `REPLACE_WITH_ID_FROM_WRANGLER_KV_NAMESPACE_CREATE` → production `id`
- `REPLACE_WITH_PREVIEW_ID_FROM_WRANGLER_KV_NAMESPACE_CREATE_PREVIEW` → `preview_id`
- `account_id` → your own Cloudflare account ID (shown after login)

Then deploy:
```bash
npx wrangler deploy
# → note your URL: https://crosspoint-calendar.<subdomain>.workers.dev
```

Optional secrets for real calendar data:
```bash
npx wrangler secret put GOOGLE_CALENDAR_API_KEY
npx wrangler secret put GOOGLE_CALENDAR_ID
```

### 2. Flash CrossPoint Reader firmware with Calendar Mode

Calendar Mode (Settings → Calendar Server URL) is not in the released v1.1.1. Need to build from source — PR #408 adds this feature.

Requirements: PlatformIO CLI, USB-C cable.

```bash
# Install PlatformIO if needed
pip install platformio

# Clone and flash
git clone https://github.com/crosspoint-reader/crosspoint-reader
cd crosspoint-reader
pio run --target upload   # device connected via USB-C
```

Alternative: check https://xteink.dve.al/ for a dev build with Calendar Mode.

### 3. Configure device

After deploy + firmware flash:
- Settings → Calendar Server URL → `https://crosspoint-calendar.<subdomain>.workers.dev`
- Settings → Calendar Mode → ON
