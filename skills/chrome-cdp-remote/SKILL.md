---
name: chrome-cdp-remote
description: Interact with a REMOTE Chrome browser session over Chrome DevTools Protocol (only on explicit user approval after being asked to inspect, debug, or interact with a page open in a remote Chrome). Use when the user already has a Chrome instance running elsewhere with CDP exposed and gives you its URL, as opposed to a Chrome running on this machine.
---

# Chrome CDP (Remote)

Lightweight Chrome DevTools Protocol CLI for a Chrome that is **already running elsewhere** with CDP listening on a known host:port. Connects directly via WebSocket — no Puppeteer, works with 100+ tabs, instant connection. Same command set as the local `chrome-cdp` skill, minus the local browser auto-discovery, plus remote-endpoint config and connectivity diagnostics.

Use this instead of `chrome-cdp` whenever the user gives you a CDP URL / host:port for a browser that isn't on this machine (a VM, a container, a remote dev box, a cloud browser, etc.).

## Configuring the endpoint

Set one of these environment variables before running commands (export once per shell, or prefix each invocation):

```bash
export CDP_URL=http://192.168.1.50:9222      # preferred: full base URL of the CDP HTTP endpoint
# or
export CDP_HOST=192.168.1.50                  # CDP_PORT defaults to 9222 if unset
export CDP_PORT=9222
```

`CDP_URL` can also be a direct browser-level `ws://host:port/devtools/browser/<id>` endpoint if you already have one (skips the `/json/version` lookup).

Daemons and the page-list cache are namespaced per endpoint (hashed from `CDP_URL`/`CDP_HOST:CDP_PORT`), so you can point the CLI at several different remote Chromes in the same session without their tab sessions colliding.

## Prerequisites on the remote machine

The remote Chrome must be started with remote debugging exposed to the network, not just the default loopback-only mode:

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --remote-debugging-address=0.0.0.0 \
  --remote-allow-origins=*
```

- `--remote-debugging-address=0.0.0.0` — binds beyond localhost so it's reachable from this machine.
- `--remote-allow-origins=*` — **required on Chrome 111+**. Chrome added DevTools-origin validation that silently rejects the WebSocket handshake from a different host without this (or a matching explicit origin). This is the most common cause of a hang/timeout that isn't a network issue.
- If you can't (or don't want to) expose the port publicly, tunnel it instead and treat it as local: `ssh -L 9222:localhost:9222 user@remote-host`, then `CDP_URL=http://127.0.0.1:9222`.
- Node.js 22+ on this machine (uses built-in `WebSocket`/`fetch`).

If the user says CDP is "already on" — this is informational; only use it to diagnose a connection failure.

## Commands

All commands use `scripts/cdp-remote.mjs`. The `<target>` is a **unique** targetId prefix from `list`; copy the full prefix shown in the `list` output (for example `6BE827FA`). The CLI rejects ambiguous prefixes.

### List open pages

```bash
CDP_URL=http://192.168.1.50:9222 scripts/cdp-remote.mjs list
```

If this fails, the error message includes a checklist (reachability, `--remote-allow-origins`, tunneling) — read it before retrying blindly.

### Take a screenshot

```bash
scripts/cdp-remote.mjs shot <target> [file]    # default: screenshot-<target>.png in runtime dir
```

Captures the **viewport only**. Scroll first with `eval` if you need content below the fold. Output includes the page's DPR and coordinate conversion hint (see **Coordinates** below).

### Accessibility tree snapshot

```bash
scripts/cdp-remote.mjs snap <target>
```

### Evaluate JavaScript

```bash
scripts/cdp-remote.mjs eval <target> <expr>
```

> **Watch out:** avoid index-based selection (`querySelectorAll(...)[i]`) across multiple `eval` calls when the DOM can change between them (e.g. after clicking Ignore, card indices shift). Collect all data in one `eval` or use stable selectors.

### Other commands

```bash
scripts/cdp-remote.mjs html    <target> [selector]   # full page or element HTML
scripts/cdp-remote.mjs nav     <target> <url>         # navigate and wait for load
scripts/cdp-remote.mjs net     <target>               # resource timing entries
scripts/cdp-remote.mjs click   <target> <selector>    # click element by CSS selector
scripts/cdp-remote.mjs clickxy <target> <x> <y>       # click at CSS pixel coords
scripts/cdp-remote.mjs type    <target> <text>         # Input.insertText at current focus; works in cross-origin iframes unlike eval
scripts/cdp-remote.mjs loadall <target> <selector> [ms]  # click "load more" until gone (default 1500ms between clicks)
scripts/cdp-remote.mjs evalraw <target> <method> [json]  # raw CDP command passthrough
scripts/cdp-remote.mjs open    [url]                  # open new tab
scripts/cdp-remote.mjs close   <target>               # close a tab in the remote browser
scripts/cdp-remote.mjs stop    [target]               # stop local daemon(s) only
```

> **`stop` vs `close`:** `stop` only ends this CLI's local session (kills the per-tab daemon) — the tab stays open in the remote browser. `close` actually closes the tab remotely (`Target.closeTarget`). After a task that opened a tab (e.g. via `open`), close it with `close <target>` when done — otherwise it accumulates as an orphaned tab in the remote Chrome.

## Coordinates

`shot` saves an image at native resolution: image pixels = CSS pixels × DPR. CDP Input events (`clickxy` etc.) take **CSS pixels**.

```
CSS px = screenshot image px / DPR
```

`shot` prints the DPR for the current page. Typical Retina (DPR=2): divide screenshot coords by 2.

## Tips

- Prefer `snap` over `html` for page structure — much smaller and semantic.
- Use `type` (not eval) to enter text in cross-origin iframes — `click`/`clickxy` to focus first, then `type`.
- A background daemon per tab keeps the CDP session open across commands (important over a WAN — avoids a fresh WebSocket handshake per command). Daemons auto-exit after 20 minutes of inactivity, or run `stop` to end them explicitly.
- Unlike the local `chrome-cdp` skill, there's no "Allow debugging" modal to click through — remote Chrome started with `--remote-debugging-port` at launch doesn't show one. If a daemon fails to start, it's a connectivity/config issue, not a pending approval dialog (see Prerequisites).
