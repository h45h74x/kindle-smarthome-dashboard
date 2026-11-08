# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A Kindle smarthome dashboard for Home Assistant (HA). Two independent parts, connected by a WebSocket on port 4365. Development devices: Kindle Paperwhite 5th gen (EY21) and Kindle Paperwhite 12th gen. The original Paperwhite 2 (6th gen) was also supported. The README says this is a quick-and-dirty, hardcoded personal project (German UI strings, personal calendar/entity IDs).

There is no build, lint or test setup (`npm test` is a stub).

## Commands

Proxy (from `websocket-proxy/`; ES modules, Node with global `WebSocket`, so Node 22+):
- `cp config.sample.json config.json`, fill in values, then `node main.js` (or `npm start`)
- `SKIP_LIB_INSTALL=1 node main.js` skips the auto `npm install` of module `libDeps`

Kindle app: no build. Copy `smarthomedisplay/` to `/mnt/us/extensions/` on a jailbroken Kindle and copy `mesquite/config.sample.js` to `mesquite/config.js`. `mesquite/index.html` also opens in a desktop browser for debugging. It shows a gray frame when `window.kindle` is absent.

## Architecture

**Kindle side** (`smarthomedisplay/`): a KUAL extension.
- `bin/run.sh` stops the Kindle GUI, disables the screensaver, starts Utild (lets the app run shell commands via `kindle.messaging`, used for backlight), registers a `mesquite` app in `appreg.db`, launches it, and restores the GUI on power-button press.
- `mesquite/` is the dashboard. It is a plain old-WebKit page (the Kindle's engine), so write ES5 only (`var`, function expressions, no modules). `js/index.js` is the single script. `config.js` (gitignored) supplies globals such as `WS_URL` and `DISPLAY_CALENDARS`.

**Proxy side** (`websocket-proxy/`): `main.js` is a tiny module loader.
- It loads every `modules/*.module.js` in filename order. The numeric prefix is the only dependency ordering (there is a TODO for real sorting), so `00-` loads before `99-`.
- The first line of each module must be a JSON comment, e.g. `// { "id": "kindle-display", "libDeps": [...], "modDeps": [...] }`. The loader parses it and `npm install`s missing `libDeps`. A module whose `modDeps` are not loaded yet fails to load.
- Module config comes from `config.json[meta.id]`. Modules extend `HAModule` and implement `start()`. Other modules are reached via `main.loadedModules.<id>`. `main.persistentStorage` is saved to `persistentStorage.json`.
- `00-homeassistant`: client for the HA WebSocket API (auth, ping heartbeat, reconnect), emits raw HA messages and exposes `sendMessage` with an auto-incrementing `messageId`. It also has `getRequest` for the REST API.
- `99-kindle-display`: the WebSocket server (faye-websocket) for Kindles. It exists because the Kindle only speaks an old WebSocket version. Kindle message types:
  - `init`: subscribes to entities and events.
  - `call_service`.
  - `fire_event`.
  - `fetch_history`: hourly statistics, with gaps zero-filled.
  - `fetch_calendars`: REST calendar queries, with dates normalized to ms timestamps.
  
  Auth is the `accessToken` query param. Subscriptions are tracked per connection by HA message id (`meta.entitySubscriptionId`, `meta.subscribedEvents`) and re-sent on HA `auth_ok` after a reconnect. Heavy processing lives here so the Kindle stays light.

Adding a Kindle feature usually touches both sides: a new message type in `99-kindle-display.module.js` and a matching handler in `mesquite/js/index.js`.

## Developing for the Kindle (low-power e-ink device)

- Keep the Kindle thin: do filtering, gap-filling, date normalization and aggregation in the proxy and send small, ready-to-render payloads. Subscribe only to the entities and events the page needs.
- Use ES5 and old-WebKit CSS. No modern JS or CSS features, and no frameworks.
- E-ink has slow refresh and ghosting. Avoid animations and transitions (the chart uses `animation: { duration: 0 }`). Update the DOM only on real changes, and prefer small targeted updates over re-rendering whole lists. `refreshScreen()` (a white flash) clears ghosting.
- Keep timers few and slow. Weather polls every 2h, and the clock reschedules itself to the next minute boundary rather than polling each second.
- Keep the connection cheap but robust: the app-level `ping`/`pong` heartbeat detects dead sockets, and on close the app reconnects after 10s. The proxy re-subscribes to HA after reconnects.
- Prefer small SVG icons and the already-bundled `chart.min.js`. Avoid large images and new heavy libraries.
- Backlight is set by writing to sysfs via Utild (`kindle.messaging`). Keep it dim or off by default to save battery and respect night mode.
- Because the app prevents standby, the battery drains fast; the README describes a battery-less power mod.
