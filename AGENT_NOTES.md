# Agent handoff: `music-assistant.html` (Mavis Media Center) in Home Assistant

Audience: the coding agent working on Jermaine's local machine / Home Assistant box.
Subject file: **`music-assistant.html`** on branch `claude/music-assistant-html-replica-0cn58p`.

> **IMPORTANT — this version supersedes any locally modified copy.** The live
> Music Assistant client is now FULLY IMPLEMENTED in this file and verified
> end-to-end against a protocol mock (handshake, auth, players, queues, library,
> artwork, search, transport, events). If you previously edited a deployed copy,
> discard those edits and redeploy this file fresh; the only thing to carry over
> is any Home Assistant device wiring, which now belongs in the `haAction()`
> function (see §4).

## 1. What already works (do not reimplement)

With a reachable Music Assistant server configured, the page live-loads:

- **Album shelves**: 12 tiles from `music/albums/library_items`, with real cover
  art via MA's image proxy overlaid on the placeholder SVGs. Chips switch the
  query (newest / most played / random / favorites); the right-arrow pages by 12.
- **Search**: the search bar debounces into `music/search` (albums) and fills the
  shelves; clearing restores the library view.
- **Rooms & Zones**: tiles are populated from `players/all` with real player
  names; EQ bars animate for players whose `state == "playing"`; clicking a tile
  switches which player/queue every control targets. Extra tiles beyond the real
  player count stay decorative.
- **Now Playing**: title / artist / album / artwork / duration from
  `player_queues/all` + `queue_updated` events; elapsed time from
  `queue_time_updated` events; play state, shuffle, repeat reflected live.
- **Transport**: play/pause/next/previous (`players/cmd/*`), volume
  (`players/cmd/volume_set`), seek (`players/cmd/seek`), shuffle/repeat
  (`player_queues/shuffle` / `repeat`), album click → `player_queues/play_media`
  on the active queue, heart → `music/favorites/add_item`.
- **Demo fallback**: with no host configured the page renders the built-in
  placeholder content and logs `[MA demo]` instead of erroring. Keep this.

## 2. Configuration (no code edits, no committed secrets)

Open the deployed page once with query params — they persist to localStorage:

```
http://<HA>:8123/local/mavis/index.html?host=homeassistant.local&port=8095
```

Add `&token=<long-lived MA token>` if the MA server requires auth, and
`&tls=true` if MA is behind TLS. `?host=clear` wipes the stored config.
The repo copy must keep empty defaults (public GitHub Pages site).

## 3. Mounting in the HA dashboard

1. `cp music-assistant.html /config/www/mavis/index.html`
2. Settings → Dashboards → **Add dashboard → Webpage** →
   `/local/mavis/index.html` (or classic `panel_iframe` in `configuration.yaml`).
3. Open it once with the `?host=...` params from §2.

Gotchas:
- **Mixed content**: if HA is served over HTTPS, the browser will block a
  `ws://` connection to MA. Options: access HA over plain http on the LAN,
  put MA behind TLS and use `&tls=true`, or proxy `/ws` through the same origin.
- The layout is fixed-width by design (`min-width: 1580px`); smaller screens
  scroll. Do not make it responsive without being asked.

## 4. Your remaining work

1. **Deploy + configure** per §2–3, then open the browser console. Every MA
   call that fails logs `[MA] <command> failed: ... — verify the command name
   on /api-docs`. Command names follow MA 2.x conventions and were verified
   against a protocol mock, not a real server — if any log appears, check the
   exact name/args at `http://<MA_HOST>:8095/api-docs` and adjust that one call.
2. **Wire the non-music controls** — device cards, bottom pills, Quick Actions
   (`tv_*`, `avr_*`, `lights_*`, `party_mode`, `all_off`, …). These are
   **Home Assistant** entities/scripts/scenes, NOT Music Assistant. They all
   funnel into the single `haAction(cmd)` function near the end of the file —
   implement it against the HA WebSocket API (`wss://<HA>/api/websocket`,
   auth with a long-lived token, then `call_service`), or leave individual
   commands as logged placeholders until entities are chosen.
3. Optional polish, only if asked: un-favorite support (needs the library item
   id), dedicated Albums/Artists/Playlists views for the sidebar nav, group
   room creation (`players/create_group` family).

## 5. Hard constraints (unchanged)

- **No visual redesign.** Layout, colors, spacing, typography, copy are the
  deliverable. Compare against `reference-render.png` (1672×941) after changes.
- **No renamed/removed hooks**: `data-album`, `data-room`, `data-cmd`,
  `data-action`, `data-nav`, `data-device`, and the `np-*` / `btn-*` ids.
- **No frameworks, bundlers, or npm dependencies.** Single file (at most split
  into `mavis.css` / `mavis.js` alongside).
- **No secrets in the repo.** Tokens live in localStorage / the deployed copy only.
- **Keep demo mode** and the SVG artwork fallbacks (live art overlays them;
  a broken image URL automatically falls back to the SVG).
- **Don't touch** the slider fill mechanism, EQ animation CSS, or live clock.
- Sliders must never send commands during initial paint — only on user input
  (already handled in `bindRange`; preserve that behavior).

## 6. Music Assistant protocol reference (as implemented)

- Endpoint `ws(s)://<host>:<port>/ws`. First server frame is ServerInfo
  (`server_version`, `schema_version`, `base_url`).
- If a token is configured the client sends `{command: "auth", args: {token}}`
  and proceeds even if the server doesn't require it.
- Requests: `{message_id, command, args}` → `{message_id, result}` or
  `{message_id, error_code, details}`. 10s client-side timeout per command.
- Events: `{event, object_id, data}` — handled: `player_updated`,
  `player_added`, `queue_updated`, `queue_items_updated`, `queue_time_updated`.
- Artwork: item `metadata.images[0]`; if not `remotely_accessible`, the URL is
  `{base_url}/imageproxy?path=...&provider=...&size=...` (see `imageUrl()`).

## 7. Testing without the real server

`mock-ma.mjs` (in the delivered zip) is a Node mock of the MA WebSocket API
(`npm i ws && node mock-ma.mjs`, then open the page with
`?host=127.0.0.1&port=8096`). It serves fake players, a queue, a 24-album
library, and search results, and pushes a `queue_time_updated` event — useful
for regression-testing UI changes without touching the production server.
