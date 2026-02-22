---
summary: "Per-service browser automation playbooks for managed profiles and remote CDP"
read_when:
  - Setting up browser automation for SaaS services
  - Choosing between local managed profiles and remote CDP attach
  - Handling Comet or hosted browser CDP compatibility gaps
title: "Browser workflows for SaaS"
---

# Browser workflows for SaaS

Use this guide to standardize browser automation across services while keeping sessions isolated and repeatable.

## Service profile setup patterns

Set one stable profile name per service (for example `github`, `linear`, `figma`, `comet`) under `browser.profiles`.

### Pattern A: local managed browser profile

Use this when OpenClaw should launch and manage the browser directly.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "github",
    profiles: {
      github: { cdpPort: 18810, color: "#24292F" },
      linear: { cdpPort: 18811, color: "#5E6AD2" },
      figma: { cdpPort: 18812, color: "#A259FF" },
    },
  },
}
```

### Pattern B: remote CDP attach

Use this when a remote browser runtime exposes CDP endpoints.

```json5
{
  browser: {
    enabled: true,
    profiles: {
      comet: { cdpUrl: "https://cdp.example.internal", color: "#FF6A00" },
      browserless: { cdpUrl: "https://production-sfo.browserless.io", color: "#00A36C" },
    },
  },
}
```

Use per-service profile names even for remote CDP so automation prompts and scripts remain deterministic.

## Login, session persistence, and credential safety

Recommended strategy:

1. Bootstrap each profile once with manual login and MFA.
2. Reuse the same profile name for every future run.
3. Re-auth only when session expiration or policy changes require it.

Quick bootstrap sequence:

```bash
openclaw browser --browser-profile github start
openclaw browser --browser-profile github open "https://github.com/login"
```

Then complete login interactively and verify persistence:

```bash
openclaw browser --browser-profile github stop
openclaw browser --browser-profile github start
openclaw browser --browser-profile github open "https://github.com"
```

Security notes:

- Browser profile data can include active cookies, tokens, and local storage state.
- Keep profile directories and config files private to your user account.
- Do not commit authenticated `cdpUrl` values, tokens, or exported cookie data.
- Prefer environment variables or a secrets manager for remote CDP credentials.
- Restrict remote CDP endpoints to private networking (loopback, VPN, or tailnet).

## Per-service runbooks

Use this checklist for each service run.

### 1) Open app

```bash
openclaw browser --browser-profile linear open "https://linear.app"
```

### 2) Navigate to project or context

```bash
openclaw browser --browser-profile linear navigate "https://linear.app/openclaw/team/ENG"
openclaw browser --browser-profile linear snapshot --interactive
```

### 3) Upload or download assets

```bash
openclaw browser --browser-profile linear upload ./artifacts/release-notes.pdf
openclaw browser --browser-profile linear downloads --json
```

### 4) Verify completion

Use UI plus network verification together.

```bash
openclaw browser --browser-profile linear snapshot --interactive --json
openclaw browser --browser-profile linear requests --filter api --json
openclaw browser --browser-profile linear screenshot --full-page
```

Verification signals to capture:

- UI confirmation text or status badge in snapshot output.
- Expected API status codes or endpoint calls in `requests` output.
- Final screenshot artifact for audit trails.

## Comet compatibility checks and fallback

Some Comet deployments do not expose complete CDP discovery and websocket endpoints.

### Compatibility check

Run both endpoint checks before using a remote Comet profile:

```bash
curl -fsS "http://127.0.0.1:9222/json/version"
curl -fsS "http://127.0.0.1:9222/json/list"
```

Treat CDP attach as unavailable when:

- Either command times out or fails.
- Responses are not valid DevTools JSON payloads.
- WebSocket connect repeatedly fails during OpenClaw profile startup.

### Fallback path: managed Chromium profile

Switch the service to a local managed profile and continue automation:

```json5
{
  browser: {
    profiles: {
      "comet-managed": { cdpPort: 18820, color: "#FF6A00" },
    },
  },
}
```

```bash
openclaw browser --browser-profile comet-managed start
openclaw browser --browser-profile comet-managed open "https://comet.example"
```

This fallback preserves session persistence and avoids dependency on remote CDP attach.

## Troubleshooting matrix

| Failure                            | Likely cause                                              | Check                                                               | Fix                                                                        |
| ---------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Auth wall after restart            | Session cookie expired or profile mismatch                | Confirm correct `--browser-profile`; run `snapshot` on landing page | Re-login on the same profile, then retest persistence by restart           |
| Selector drift or missing target   | UI changed and old selector/ref is stale                  | Run fresh `snapshot --interactive` and inspect refs                 | Replace brittle selector usage with current snapshot refs                  |
| Upload blocked or no file attached | Hidden file input, permission prompt, or invalid path     | Verify path exists locally; inspect page errors and snapshot        | Use `upload` after focusing the right context, then confirm via UI/network |
| CDP disconnect during run          | Remote endpoint unstable, idle timeout, or network policy | Check `/json/version`, `/json/list`, and retry profile start        | Reconnect; if repeated, move workflow to managed local profile             |

## See also

- [Browser](/tools/browser)
- [Browser login + X or Twitter posting](/tools/browser-login)
- [Browser troubleshooting](/tools/browser-linux-troubleshooting)
