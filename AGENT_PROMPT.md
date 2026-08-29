# Copy-paste prompt for the local coding agent

Paste everything below the line into the agent working on the Home Assistant box.

---

I'm giving you a new version of the Mavis Media Center dashboard
(`music-assistant.html`, from the `mavis-media-center` package / the
`claude/music-assistant-html-replica-0cn58p` branch of jeri2good/jeri2good.github.io).

**Start by discarding every local edit you made to the previously deployed copy.**
Your earlier changes had real bugs — among them, the no-token path never marked the
WebSocket connection ready, which is why everything stayed stuck in demo mode. The new
file already contains a complete, working Music Assistant client that was verified
end-to-end against a protocol mock: album shelves with real artwork, library search,
player/room enumeration, live Now Playing with events, and all transport controls
(play/pause/next/previous, volume, seek, shuffle, repeat, album-click-to-play,
favorites). Do not reimplement, refactor, or "improve" any of that. Read
`AGENT_NOTES.md` in full before touching anything.

Your tasks, in order:

1. Deploy the new file exactly as-is:
   `cp music-assistant.html /config/www/mavis/index.html`
   and confirm the dashboard entry (Settings → Dashboards → Webpage, or
   `panel_iframe`) points at `/local/mavis/index.html`.

2. Configure it with URL params — do NOT edit code or hardcode secrets:
   open `http://<HA>:8123/local/mavis/index.html?host=<MA_HOST>&port=8095`
   once (add `&token=<MA long-lived token>` only if the Music Assistant server
   requires auth; `&tls=true` only if MA is behind TLS). The values persist in
   localStorage. If Home Assistant is served over HTTPS, a plain `ws://`
   connection will be blocked as mixed content — solve that by accessing HA over
   http on the LAN or proxying, not by rewriting the client.

3. Open the browser console and verify the live data loads. The client logs
   every failure as `[MA] <command> failed: ... — verify the command name on
   /api-docs`. For each such line (there may be none), look up the exact
   command name and argument shape at `http://<MA_HOST>:8095/api-docs` and fix
   only that one call site. Do not restructure the client.

4. Wire the non-music controls. The device cards, bottom pills, and Quick
   Actions (`tv_*`, `avr_*`, `spk_*`, `proj_*`, `lights_*`, `source_*`,
   `party_mode`, `relax_mode`, `good_night`, `all_off`, `multi_room_sync`,
   `sound_mode`, `night_mode`, `audio_delay`, `output_routing`) are Home
   Assistant territory — scripts, scenes, remote/light/media_player entities —
   NOT Music Assistant. They all funnel into the single `haAction(cmd)` function
   near the end of the file. Implement it against the HA WebSocket API
   (`ws(s)://<HA>/api/websocket` → `auth` with a long-lived token →
   `call_service`), mapping each command string to a service call. Ask me which
   entities/scripts to use for any mapping you can't determine; leave unmapped
   commands as logged placeholders rather than guessing.

Hard constraints — violating any of these is a failed task:
- Zero visual changes. After your work, the page must still match
  `reference-render.png` (1672×941) pixel-for-pixel apart from live data.
- Do not rename or remove the `data-*` hooks or `np-*`/`btn-*` ids.
- No frameworks, bundlers, or npm dependencies in the page.
- No tokens or hostnames committed anywhere; runtime config only.
- Keep demo mode (no host configured → placeholder content, `[MA demo]` logs,
  zero errors) and keep the SVG artwork fallbacks under the live images.
- Sliders must never send commands during initial paint — only on user input.

To regression-test without touching the real server: `npm i ws && node
mock-ma.mjs`, then open the page with `?host=127.0.0.1&port=8096`. You should
see the mock's albums, players, and Now Playing data appear.

Definition of done: on the real server, the shelves show my actual library with
real cover art, search works, room tiles show my actual players, Now Playing
tracks what's really playing and updates live, every transport control moves the
real player, the §4 device controls drive their HA entities (or are explicitly
listed back to me as awaiting entity choices), and the page still looks
identical to the reference render.
