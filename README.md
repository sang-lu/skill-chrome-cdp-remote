# chrome-cdp-remote

An Agent Skill that lets a coding agent drive a Chrome instance running on a **different machine** — a VM, a container, a remote dev box, a cloud browser — over the raw Chrome DevTools Protocol (WebSocket). No Puppeteer, no local browser: it connects directly to a CDP endpoint you point it at.

The agent can list open tabs, take screenshots, read the accessibility tree, evaluate JS, click, type, navigate, and close tabs — all against a browser it doesn't run itself.

See [`skills/chrome-cdp-remote/SKILL.md`](skills/chrome-cdp-remote/SKILL.md) for the full command reference.

## Prerequisites: launching the remote Chrome

The remote machine must start Chrome with debugging exposed to the network — the default (loopback-only) `--remote-debugging-port` is not reachable from another machine:

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --remote-debugging-address=0.0.0.0 \
  --remote-allow-origins=*
```

- `--remote-debugging-address=0.0.0.0` — binds beyond localhost so this machine can reach it.
- `--remote-allow-origins=*` — **required on Chrome 111+**. Without it, Chrome silently rejects the WebSocket handshake from a different host — the most common cause of a hang/timeout that looks like a network problem but isn't.
- If you don't want to expose the port publicly, tunnel it instead and treat it as local: `ssh -L 9222:localhost:9222 user@remote-host`, then point the skill at `CDP_URL=http://127.0.0.1:9222`.
- Node.js 22+ is required on the machine running the agent (the skill uses Node's built-in `WebSocket`/`fetch`, no dependencies).

Once Chrome is up, tell the agent the endpoint (e.g. `CDP_URL=http://192.168.1.50:9222`) and ask it to list tabs, screenshot a page, etc.
