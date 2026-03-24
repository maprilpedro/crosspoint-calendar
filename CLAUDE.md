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
- Location: Brooklyn, NY (40.7243°N, 73.9493°W)
- Timezone: `America/New_York`
- Weather cache TTL: 15 min (errors: 5 min)
