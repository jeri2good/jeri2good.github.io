# Agent handoff: wiring `music-assistant.html` into Home Assistant

Audience: the coding agent working on Jermaine's local machine / Home Assistant box.
Subject file: **`music-assistant.html`** on branch `claude/music-assistant-html-replica-0cn58p`.

This page is a pixel-faithful replica of the "Aurora Media Center" dashboard design.
It is a single self-contained HTML file (no build step, no npm, no framework; the only
external resource is the Google Fonts *Inter* stylesheet). All UI controls are already
present and carry stable `data-*` / `id` hooks, and a stubbed `MusicAssistant` WebSocket
client class sits at the bottom of the file. Your job is only to (1) mount it in the HA
dashboard and (2) replace the stubs with live data. **Do not redesign anything.**

---

## 1. Mounting it in the Home Assistant dashboard

Recommended path (simplest, no custom-card development):

1. Copy the file to HA's public www folder:
   `cp music-assistant.html /config/www/aurora/index.html`
   It is then served at `http://<HA>:8123/local/aurora/index.html`.
2. Add it as a sidebar panel in `configuration.yaml`:

   ```yaml
   panel_iframe:          # or the "Webpage" dashboard type in newer HA (Settings → Dashboards → Add → Webpage)
     aurora:
       title: "Aurora Media"
       icon: mdi:music-box-multiple
       url: /local/aurora/index.html
   ```

   In HA ≥2024 the UI equivalent is Settings → Dashboards → **Add dashboard → Webpage**,
   pointing at `/local/aurora/index.html`. Either way the page fills the view.
3. Reload HA (or restart) and open the panel. The page runs in "demo mode" until
   `MA_CONFIG.host` is set (see §3).

Notes on this mounting mode:
- The layout is designed for ≥1580px wide (`.app { min-width: 1580px }`). On a wall
  tablet at a smaller resolution the page scrolls horizontally; that is intentional to
  preserve the design. Do not "make it responsive" without being asked.
- If mixed-content blocking bites (HA over HTTPS, MA WebSocket over ws://), serve HA and
  the MA websocket from the same origin via an HA ingress/proxy, or use the HA-API
  fallback in §4 instead of a direct MA socket.

## 2. What NOT to do (hard constraints)

- **Do not change the visual design.** Layout, colors, spacing, typography, icon set,
  copy, and the fixed three-column structure are the deliverable. Any diff should be
  functional, not cosmetic.
- **Do not rename or remove hooks.** All wiring points are `data-album`, `data-room`,
  `data-group`, `data-cmd`, `data-action`, `data-nav`, `data-device` attributes and the
  `np-*` / `btn-*` element ids. Other code (and future sessions) target these.
- **Do not add a framework, bundler, or npm dependency.** Keep it one file. If you must
  split, `aurora.css` + `aurora.js` next to the HTML is the maximum allowed.
- **Do not commit secrets.** `MA_CONFIG.token` / HA long-lived tokens must never be
  committed — this repo is a public GitHub Pages site. Inject them locally (the copy in
  `/config/www` is fine to configure; the repo copy keeps empty strings).
- **Do not remove demo mode.** With no `host` configured the page must keep rendering
  with its placeholder content and log `[MA demo]` instead of throwing.
- **Do not replace the inline-SVG album art wholesale.** When live data arrives, swap
  art by setting a background/`<img>` from the item's `image_url`; keep the SVGs as the
  no-artwork fallback.
- **Do not touch** the range-slider "filled track" mechanism (`.filled` + `--fill`
  custom property, updated in `bindRange()`), the EQ-bar animation CSS, or the live
  clock (`tickClock`).
- The static text in device cards / bottom pills (Samsung QN90B, Denon AVR-X3800H,
  "Apple TV 4K", "Movie", "0 ms"…) are placeholders for **Home Assistant entities, not
  Music Assistant** — wire them per §5, or leave them static for now.

## 3. Music Assistant server API — exactly how this page ties in

Music Assistant (the HA add-on) exposes its own WebSocket API, separate from HA's:

- **Endpoint:** `ws://<MA_HOST>:8095/ws` (same server that hosts the MA web UI).
  Interactive, auto-generated command reference: `http://<MA_HOST>:8095/api-docs` —
  **verify every command name there before wiring; treat the names below as the map,
  not gospel** (they follow MA 2.x conventions).
- **Handshake:** on connect the server pushes a `ServerInfo` message. If authentication
  is enabled, send the auth command with a token before anything else; on success the
  client is auto-subscribed to the event stream.
- **Command format (JSON-RPC style):**
  ```json
  { "message_id": 7, "command": "players/cmd/play", "args": { "player_id": "..." } }
  ```
  Responses echo `message_id` with a result or error; events arrive unsolicited.

In the page, this is implemented by:

| Code | Location |
|---|---|
| `MA_CONFIG` (host / port / token / defaultPlayer) | `music-assistant.html:1360` |
| `MusicAssistant` class (connect, auto-reconnect, `command()`, `playerCmd()`) | `music-assistant.html:1367` |
| `handleMessage()` — currently a TODO; route results/events into the UI here | inside the class |
| `activeRoom` — the player/queue every transport control targets | `music-assistant.html:1406` |

### UI element → MA command map

| UI hook | MA command (verify on `/api-docs`) | args |
|---|---|---|
| `#btn-play` (`music-assistant.html:1304`) | `players/cmd/play` / `players/cmd/pause` (or `players/cmd/play_pause`) | `player_id` |
| `#btn-next` / `#btn-prev` | `players/cmd/next` / `players/cmd/previous` | `player_id` |
| `#np-volume` slider | `players/cmd/volume_set` | `player_id`, `volume_level` 0–100 |
| `#np-seek` slider | `players/cmd/seek` (or `player_queues/seek`) | `player_id`/`queue_id`, `position` seconds |
| `#btn-shuffle` | `player_queues/shuffle` | `queue_id`, `shuffle_enabled` bool |
| `#btn-repeat` | `player_queues/repeat` | `queue_id`, `repeat_mode` `off`/`one`/`all` |
| album card click (`.album-card[data-album]`) | `player_queues/play_media` | `queue_id`, `media` = item uri, `option` play/replace/next |
| album shelves + `#row-next` pager | `music/albums/library_items` | `limit`, `offset`, `order_by` (chips map to sorts: New Releases → recent, Top Albums → play count, My Favorites → `favorite: true`) |
| sidebar Artists / Playlists / Genres / Recent | `music/artists/library_items`, `music/playlists/library_items`, `music/tracks/library_items`, recently-played endpoint | same paging pattern |
| `#search-input` | `music/search` | `search_query`, `media_types`, `limit` |
| room cards (`.room-card[data-room]`) | `players/all` to enumerate real players; store the chosen `player_id` in `activeRoom` | — |
| `#np-like` heart | `music/favorites/add_item` / `remove_item` | item uri |
| group card / Whole House / Manage Groups | player-group commands (`players/cmd/group` family / `players/create_group`) | verify exact names on `/api-docs` |

### State in (events → UI)

Subscribe (automatic after auth) and handle in `handleMessage()`:

- `player_updated` → volume (`#np-volume`, `#vol-val`), play state (`setPlaying()`),
  which `.room-card` gets the `playing` class / bright EQ bars.
- `queue_updated` / `queue_items_updated` → Now Playing panel: `#np-title`, `#np-artist`,
  `#np-album` from `queue.current_item.media_item` (`.name`, `.artists[].name`,
  `.album.name`), duration → `#np-duration` and `#np-seek.max`.
- `queue_time_updated` → `#np-elapsed` + seek position (then **remove the demo
  progress ticker**, the `setInterval` block labeled "demo progress ticker").
- Artwork: each media item exposes image metadata (and MA serves an image proxy);
  set the resolved URL onto `.np-art` and the matching `.album-art`.

The mapping from the six replica room names to real `player_id`s should be a small
config table at the top of the file next to `MA_CONFIG`, populated from `players/all`.

## 4. Fallback: drive it through Home Assistant instead of MA directly

Every MA player is also an HA `media_player` entity. If the direct socket is awkward
(auth/CORS/mixed content), swap the transport layer only — same UI hooks:

- HA WebSocket: `wss://<HA>/api/websocket`, auth with a long-lived access token, then
  `call_service` on `media_player.media_play_pause`, `media_next_track`,
  `volume_set` (0–1 float, divide the slider by 100), `media_seek`,
  `shuffle_set`, `repeat_set`, targeting entities like `media_player.living_room`.
- MA-specific services: `music_assistant.play_media` (accepts artist/album/track by
  name or uri), and MA's search action, for the album grid and search box.
- Subscribe to `state_changed` for those entities to fill Now Playing (attributes:
  `media_title`, `media_artist`, `media_album_name`, `media_duration`,
  `media_position`, `entity_picture` for artwork, `volume_level`).

## 5. Non-music controls (these are NOT Music Assistant)

The device cards (`data-device` / `data-cmd`: `tv_*`, `avr_*`, `spk_*`, `proj_*`,
`lights_*`, `source_*`), bottom pills (`multi_room_sync`, `source_select`,
`sound_mode`, `night_mode`, `audio_delay`, `output_routing`) and sidebar Quick Actions
(`data-action`: `party_mode`, `relax_mode`, `good_night`, `all_off`) should be wired to
**Home Assistant** scripts/scenes/entities (remote, light, switch domains) via the HA
WebSocket path in §4. They currently all funnel through `ma.command('custom/...')` as a
placeholder — replace that dispatcher, keep the hooks.

## 6. Definition of done

- Page loads inside the HA dashboard, looks byte-for-byte like the current replica.
- With `MA_CONFIG.host` set: transport controls, volume, seek, shuffle/repeat operate
  the selected room's real player; Now Playing reflects live state within ~1s; album
  shelves and search show the real MA library with real artwork; room cards list real
  players and switch the control target.
- With no host configured: identical behavior to today's demo mode, zero console errors.
- No secrets in the repo; no visual regressions (compare against a fresh screenshot at
  1672×941).
