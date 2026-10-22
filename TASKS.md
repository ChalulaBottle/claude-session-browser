# TASKS — MCP server + agent bridge for Claude Session Browser

## RESUME STATE (update after every increment, then commit)

- **Status:** Scoping done, nothing built yet. Fork is set up and running locally
  (`ChalulaBottle/claude-session-browser`, cloned to
  `Temp Claude Session Browser/`), deps installed, verified live against 35 real
  sessions (see `docs/screenshot-sessions-live.png` — contains real session
  titles, don't publish it publicly without checking with the operator first).
- **Next exact action:** start on Phase 1 — expose the existing `Api` class
  (in `claude_sessions.py`, ~line 3318) over a local HTTP server, since every
  read/resume/tag capability the operator asked for already exists as a Python
  method on that class today. No new session-browsing logic needs writing —
  this is a transport/wrapper job, not a rewrite.
- **Not started:** Clawdmeter side (hardware not arrived yet — operator is
  ordering the ESP32-C6-Touch-AMOLED-2.16" specifically, the board the
  `csb-buddy` buddy-mirroring fork targets). MCP server itself. Anything
  touching the voice/whisper project (mentioned as a future extension target,
  not scoped yet).

---

## What was decided (operator's own answers, 2026-09-20)

- **Target:** the Session Browser and the Clawdmeter device, treated as one
  connected system (the app already bridges to the device over Bluetooth).
  Explicitly flagged as extensible later to other local projects — the voice
  project and a Whisper-based dictation tool were named as likely future
  additions. Don't build anything that assumes Session Browser is the only
  client of this connection.
- **Interface:** both. An MCP server for Claude Code agents (the native way
  agents already call out to tools), with a thin local REST API underneath
  that both the MCP server and non-Claude bots (Discord bot, SANDY, etc.) can
  call directly.
- **Capabilities in scope:**
  1. Read-only: list/search sessions, current Claude state
  2. Resume a session (same action as a double-click)
  3. Control Clawd's displayed state directly (not just auto-detected)
  4. Manage titles/colors/tags programmatically

---

## Phase 1 — REST API over the existing `Api` class

The app already has everything capability 1, 2, and 4 need as plain Python
methods (`collect_sessions`, `resume_session`, the settings read/write
helpers `Api` wraps). This phase is exposing them, not building them.

- [ ] Pick the transport: a small `http.server`/`Flask`/`FastAPI` thread
      started alongside the existing pywebview main loop (same process,
      same `Api` instance — avoids a second source of truth for session
      state). Bind to `127.0.0.1` only, no external exposure.
- [ ] Route surface (draft, refine once building):
      - `GET  /sessions` — list/search, wraps `collect_sessions`
      - `GET  /sessions/{id}` — single session detail
      - `POST /sessions/{id}/resume` — wraps `resume_session`
      - `PATCH /sessions/{id}` — title/color/tag updates
      - `GET  /state` — current Claude activity state (whatever
        `BuddyController` currently derives for Clawd's animation)
- [ ] Auth: even loopback-only, add a shared-secret header — this API can
      trigger `resume_session`, which spawns a terminal. Don't leave that
      wide open to anything that can reach localhost.
- [ ] Verify each route against the real running app before calling this
      phase done — same rule as any UI work: prove it against the actual
      running instance, not just reading the code.

## Phase 2 — MCP server

- [ ] Thin MCP server that calls Phase 1's REST API (keeps the MCP layer
      dumb — all real logic stays in one place, the REST API).
- [ ] Tool surface mirrors the REST routes: list sessions, resume session,
      set Clawd state, update session metadata.
- [ ] Register it so Claude Code agents can actually reach it — figure out
      whether this rides the existing `agent-bridge` git message bus
      pattern or is a standalone local MCP server declared per-project.
      **Open question, needs operator input before building this part.**

## Phase 3 — control Clawd's state directly

- [ ] Today `BuddyController` infers state from tailing the newest
      transcript. Capability 3 wants agents to SET state directly instead.
      Needs a way for an external call to override the inferred state
      (temporarily, with a timeout so it doesn't get stuck showing a stale
      state if the agent that set it dies) rather than fighting the
      auto-detection loop. Design this before writing code — it's the one
      piece that doesn't already exist as a function to wrap.

## Phase 4 — Clawdmeter bridge (blocked on hardware)

- [ ] Once the ESP32-C6-Touch-AMOLED-2.16" arrives: flash stock firmware
      first (confirms the board works at all), then the `csb-buddy` branch
      from `juppeee/Clawdmeter` (adds the BLE payload field + sprites the
      buddy-mirroring needs — diffed against upstream `main` already,
      10 commits, mostly `main.cpp`/`ui.cpp`/`splash.cpp`).
- [ ] Confirm the app's existing Bluetooth bridge (`clawdmeter.py` in this
      repo, 760 lines) already talks to that fork's firmware, or needs
      updating to match whatever the BLE payload format actually is on
      the flashed firmware.
- [ ] Once flashed and paired: the Phase 3 state-override endpoint should
      reach the physical device the same way it reaches desktop Clawd —
      one state, two displays, not two separate things to control.

## Phase 5b — ECHO-branded buddy (desktop + Clawdmeter firmware)

Operator wants an ECHO-identity version of the buddy character — not a recolor of "Clawd,"
genuinely new pixel art, since Clawd's own sprites were scraped from a third-party site
(`claudepix.vercel.app`, via `Temp Clawdmeter/tools/scrape_claudepix.js` +
`tools/convert_to_c.js`, with a brand-color tint override baked in) rather than drawn
in-house. No entanglement with that source for an ECHO version — starting clean.

**Format constraints, confirmed from source (non-negotiable, both pipelines depend on them):**
- 20×20 pixels per frame, standard PNG with alpha (alpha < 128 = transparent, >= 128 = opaque)
- Max 9 opaque colors per animation + 1 transparent slot = 10-color palette per anim
- 15 animation states exist today (`idle breathe`, `idle blink`, `idle look around`,
  `work coding`, `work think`, `expression surprise`, `expression sleep`, `expression wink`,
  `dance bounce`, `dance sway`, `done`, `think`, `write`, `allow`, `limit`) — don't need all 15
  for a v1, but the state map (`BUDDY_STATE_MAP` in `claude_sessions.py`) expects these exact
  names to swap in cleanly.

- [ ] Design pass: 20×20 pixel art in ECHO's own identity (hexagon mark motif, accent
      `#35e0c0`, no colored borders per the standing house rule) for at least the core states
      that read most often: `idle breathe`, `work coding`, `work think`, `done`, `allow`,
      `limit`. Expand to the rest once the core set proves out.
- [ ] Folder convention matching `pack_sprites.py`'s expectation:
      `<root>/<anim_name_underscored>/frame_NN.png` (e.g. `work_coding/frame_00.png`).
- [ ] **Desktop path:** run `pack_sprites.py <root>` to regenerate the sprite blob. Point it
      at a new output (e.g. `echo_sprites.py`) rather than overwriting `clawd_sprites.py`
      directly, so both skins can coexist and the operator picks one in Settings.
- [ ] **Firmware path:** `convert_to_c.js` only accepts the claudepix-scraped JSON shape, not
      raw PNGs — write a small adapter (reuse `pack_sprites.py`'s PNG→palette/frame logic,
      emit `convert_to_c.js`'s C-struct output instead of the Python blob) so the SAME PNG
      folder feeds both the desktop app and `splash_animations.h` on the device. Don't
      maintain two parallel art sources.
- [ ] Wire a skin picker into Settings (desktop) so ECHO vs. default Clawd is a toggle, not a
      hard fork of the codebase.

## Phase 5 — extensibility (not scoped yet, don't build speculatively)

Named by the operator as a future direction, not a current requirement:
wiring the voice/digital-alphabet project and Whisper-based dictation work
into the same MCP server / REST API as additional capabilities. Do not
design Phase 1-3's interfaces around guesses about what that needs — revisit
this once there's an actual voice-project capability to add, and extend the
route surface then rather than over-generalizing now.
