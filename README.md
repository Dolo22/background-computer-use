# background-computer-use

An MCP server that drives other macOS apps in the background: no app activation, no window raised, no cursor moved. It ships with a skill library that teaches the model how to use it.

The server (`server/cua_server.py`) speaks plain stdio MCP, so any compliant client works: Claude Code, Claude Desktop, Cursor, Codex CLI, Goose, Continue, Zed. Its nine interaction tools (`list_apps`, `get_app_state`, `click`, `drag`, `perform_secondary_action`, `press_key`, `scroll`, `set_value`, `type_text`) wrap the macOS Accessibility, CGEvent and `CGWindowList` APIs, and six more cover desktop leases, permission checks and the cursor playground.

The skill library (`skills/`) holds markdown files with YAML frontmatter. They teach the one-snapshot-per-turn loop and carry per-app heuristics for Safari, Chrome, Clock, Numbers, Apple Music, Spotify and Notion. Clients with skill support load them automatically; for the rest, read the files into a system prompt yourself.

The project started as a parity port of the Codex desktop's built-in computer-use tools, then added animated scroll, Chrome scrolling through AppleScript JS, automatic accessibility opt-in for Electron apps, and a hard rule against silently stealing focus.

## Status

Version 0.2.0, MIT licensed. It works on macOS 15 and should work on 13 and later. The tool surface is stable. Per-app hint files get added as new quirks turn up, and `docs/APPS.md` records what works, app by app.

## Requirements

- macOS 13 or newer. 15 or newer is recommended.
- Python 3.10 or newer on your `PATH` as `python3`. Set `CUA_PYTHON` if it lives somewhere else.
- Accessibility and Screen Recording permission for the process that runs your MCP client: your terminal for a CLI client, the app bundle for Claude Desktop or Cursor. Grant both under System Settings, Privacy & Security.

The server checks the Accessibility grant on startup and names the exact bundle you need to switch on if it is missing.

## Install

### With any MCP client

Clone the repo and point your client's MCP config at `bin/cua-server`. On first run the launcher builds a virtualenv under `.venv/`, installs PyObjC, Pillow and the MCP SDK, and execs the stdio server.

```json
{
  "mcpServers": {
    "background-computer-use": {
      "command": "/absolute/path/to/background-computer-use/bin/cua-server",
      "env": {
        "CUA_PLUGIN_ROOT": "/absolute/path/to/background-computer-use"
      }
    }
  }
}
```

Config file locations:

| Client | Config path |
| --- | --- |
| Claude Code | project `.mcp.json` or `~/.claude.json` |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Cursor | `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (user) |
| Goose | `~/.config/goose/config.yaml`, different syntax |

Restart the client and the tools appear in its tool list. How a client loads the skills differs; `docs/SKILLS.md` covers each one.

### As a Claude Code plugin

```
/plugin marketplace add Dolo22/background-computer-use
/plugin install background-computer-use@background-computer-use
```

The `.claude-plugin/` directory exists only for this route. Ignore it if you use another client.

### From a clone

```bash
git clone https://github.com/Dolo22/background-computer-use.git
cd background-computer-use
claude --plugin-dir .
```

The first run takes about 15 seconds while the launcher builds the virtualenv. Later runs reuse it. `docs/DEVELOPING.md` has the environment variables, the ghost-cursor playground and troubleshooting.

## Quickstart

With the server loaded:

```
> What's on my Safari start page?
> Open https://en.wikipedia.org/wiki/Octopus in Safari.
> Scroll down to "Locomotion" in that article.
> Add 2+2 in Calculator and tell me what it says.
> Switch to the Dolphin Wikipedia tab.
```

Every one of those runs without activating the target app. Your cursor stays put.

## Ghost cursor

While an agent works, a translucent ghost cursor glides to each click target and pulses when it lands, so you can follow along without giving up your own cursor. Two styles ship: `default`, a blue arrow with a white outline, and `claude`, the Claude logo. Point `CUA_CURSOR` at any SVG/PNG file on disk to use something else.

Each MCP client gets its own ghost for the life of its server process. After about 1.5s without a tool call, the ghost drifts to the middle of the app window that client is working with, so several agents on one Mac stay visibly attached to their own app instead of stacking on the last click. The ghost exits when its server process dies.

Related variables: `CUA_CURSOR_HOTSPOT`, `CUA_CLICK_PRESS_SCALE`, `CUA_CLICK_RING`, `CUA_CLICK_OVERLAY`, `CUA_GHOST_PARK_IDLE_S`, `CUA_GHOST_HARD_IDLE_S`. `docs/DEVELOPING.md` lists the rest.

## Tools

Interaction tools:

| Tool | When to use it |
| --- | --- |
| `list_apps` | Find running apps by display name or bundle id before picking a target. |
| `get_app_state(app)` | Read the key window: accessibility tree, window screenshot, and per-app `<app_hints>`. Call it once per turn before touching an app. |
| `click(app, element_index=… \| x/y=…)` | Click an AX element or a raw coordinate. Uses `AXPress` when the element advertises it, then AppleScript, then `CGEventPostToPid`. |
| `drag(app, from_x, from_y, to_x, to_y)` | Smooth 20-step drag through `CGEventPostToPid`. |
| `perform_secondary_action(app, element_index, action)` | Run a named AX action such as `Pick`, `Increment` or `ScrollToVisible`. Refuses `ShowMenu`, `ShowAlternateUI` and `Raise` unless you pass `allow_foreground_activation=True`. |
| `press_key(app, key, element_index=…)` | xdotool-style keys: `Return`, `cmd+l`, `Page_Down`, `shift+command+t`. Can focus a specific AX element first. |
| `scroll(app, element_index, direction, pages=1, smooth=True)` | Multi-page smooth scroll. Tries `AXScroll*ByPage`, then the scroll bar's `AXValue`, then Chromium JavaScript through AppleScript, then pixel wheel events. If none of those work it raises an error instead of leaving the app focused. |
| `set_value(app, element_index, value, submit=True)` | Set an AX value. Falls back to `cmd+a`, typing and Return for Safari's URL bar and other sticky fields. |
| `type_text(app, text, element_index=…)` | Type arbitrary Unicode through synthetic key events. |

Desktop lease tools, for keeping a sequence to yourself:

| Tool | When to use it |
| --- | --- |
| `acquire_desktop(reason, ttl_s=30, wait_s=8)` | Hold the desktop across several calls so another agent cannot cut in. |
| `release_desktop()` | Give the lease back. |
| `desktop_status()` | Read-only: report the current holder. |

Diagnostics:

| Tool | When to use it |
| --- | --- |
| `check_accessibility_permission()` | Confirm the grant, and get the bundle name to switch on when it is missing. |
| `open_accessibility_settings()` | Open System Settings at Privacy & Security, Accessibility. |
| `open_cursor_playground()` | Try cursor styles without an agent attached. |

## Several agents on one Mac

A Mac has one keyboard, one mouse and one focus, so clients sharing this server take turns rather than act in parallel. Every mutating call (`click`, `drag`, `type_text`, `press_key`, `scroll`, `set_value`, `perform_secondary_action`) takes a short cross-process lease, backed by `flock` on `/tmp/cua-desktop.lock`. A second agent waits up to `CUA_LEASE_DEFAULT_WAIT_S` (8s by default), then gets `desktop busy: held by 'agent-X'`.

For work that must stay atomic, such as opening a menu, re-reading the state and picking an item, call `acquire_desktop` first and `release_desktop` afterwards. The lock drops with the process, so a crashed agent does not wedge the desktop. Set `CUA_AGENT_LABEL` in the client's env block to make contention errors readable. Read-only tools (`list_apps`, `get_app_state`, `desktop_status`) never take the lease.

## What it does and doesn't do

Background safety holds for native AppKit and SwiftUI apps and for the browsers above. It breaks down in a few places, all recorded in `docs/APPS.md` with workarounds:

- Sidebar selection in Finder, Mail, System Settings and Apple Music's library ignores clicks unless the window is key.
- An Electron app only exposes a rich tree for its key window.
- Chrome scrolling targets the scrollable ancestor at the viewport center, which can miss a sub-region or an unreachable iframe.

No tool launches an app, so the target has to be running already; `list_apps` is how you find one. macOS permission sheets belong to another process, so granting access mid-task needs you at the keyboard. Other Macs and iOS are out of scope.

## Principles

Four rules shaped the implementation:

1. Background-safe by default: no tool raises a window on its own. When an action needs the foreground, the tool says so and waits for the caller to opt in.
2. Accessibility first: `AXPress`, `AXValue` and scroll-by-page before synthetic events. Coordinates are the fallback.
3. One snapshot per turn: element indexes belong to a single `get_app_state` result, and a fresh call is cheap. Re-read after anything that changes the UI.
4. App knowledge lives in skills and `<app_hints>`: the server stays a thin wrapper over the OS, and quirks go in per-app markdown injected at call time.

## Repository layout

```
.
├── .claude-plugin/          # Claude Code plugin.json and marketplace.json
├── .mcp.json                # Claude Code per-project MCP config
├── bin/cua-server           # Launcher: builds .venv/ and execs the server
├── server/
│   ├── cua_server.py        # FastMCP stdio server (~2,200 LoC)
│   ├── cursor_ghost.py      # Ghost-cursor overlay daemon
│   ├── cursor_paths.py      # Bezier path generator
│   ├── cursor_playground.py # Ghost-cursor playground
│   ├── cursors/             # Bundled cursor images
│   ├── desktop_lease.py     # Cross-process lease
│   ├── requirements.txt
│   └── app-hints/           # Per-app <app_hints> payloads
├── skills/                  # computer-use loop, safari, chrome, clock,
│                            # numbers, apple-music, spotify, notion
├── docs/
│   ├── DESIGN.md            # Reverse-engineering notes and rationale
│   ├── APPS.md              # Per-app support matrix and limitations
│   ├── SKILLS.md            # Loading skills in each client
│   └── DEVELOPING.md        # Dev loop, env vars, troubleshooting
├── CHANGELOG.md
└── LICENSE                  # MIT
```

## Docs

Start with `docs/DESIGN.md` for how the AX, CGEvent and window APIs are used and why, then `docs/APPS.md` for the coverage matrix, `docs/SKILLS.md` for skill loading per client, and `docs/DEVELOPING.md` for local development. Changes are in `CHANGELOG.md`.

Issues and pull requests are welcome, particularly new app hints.

## License

MIT, copyright 2026 Esperance Mwamba.
