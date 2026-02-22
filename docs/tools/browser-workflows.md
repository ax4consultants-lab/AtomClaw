---
summary: "Targeted SaaS browser workflows using persistent profiles and reusable sessions"
read_when:
  - Automating SaaS dashboards with browser tooling
  - Setting up per-service browser profiles
  - Debugging remote CDP attach failures (including Comet)
title: "Browser workflows for SaaS"
---

# Browser workflows for SaaS

Use this guide when you want repeatable automation for SaaS tools (dashboards, consoles, project UIs) while keeping each service isolated.

## Per-service profile setup

Create one named profile per SaaS service under `browser.profiles.<name>`. Keep profile names stable (`github`, `linear`, `notion`, `comet`) so scripts and prompts stay deterministic.

Example:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "github",
    profiles: {
      github: { cdpPort: 18810, color: "#24292F" },
      linear: { cdpPort: 18811, color: "#5E6AD2" },
      comet: { cdpUrl: "http://127.0.0.1:9222", color: "#FF6A00" },
    },
  },
}
```

Best practices for credential persistence:

- Keep one profile per service to avoid cookie and local storage cross-talk.
- Reuse the same profile name for all automation runs so session state persists.
- Do not commit real credentials, tokens, or authenticated `cdpUrl` values to git.
- Prefer local config plus secret managers or environment variables for auth material.
- Treat the managed browser profile directories as sensitive because they can contain active sessions.

## Login and session strategy

Recommended pattern:

1. **Bootstrap manually once** per service/profile.
   - Start profile: `openclaw browser --browser-profile github start`
   - Open login page and complete auth/MFA interactively.
2. **Verify persistence** by restarting browser and confirming the app loads authenticated state.
3. **Automate reuse** in future runs using the same `--browser-profile` without re-running login flows.
4. **Refresh manually only when needed** (session expiry, password rotation, revoked device trust).

This keeps automated runs fast and avoids brittle bot-sensitive login sequences.

## End-to-end SaaS workflow examples

All examples assume an already-bootstrapped profile with a valid session.

### Open a page

```bash
openclaw browser --browser-profile linear open "https://linear.app"
```

### Navigate project or dashboard

```bash
openclaw browser --browser-profile linear navigate "https://linear.app/openclaw/team/ENG"
openclaw browser --browser-profile linear snapshot --interactive
```

### Read key UI state

Use a snapshot first, then query visible state through stable refs.

```bash
openclaw browser --browser-profile linear snapshot --interactive --json
openclaw browser --browser-profile linear evaluate "() => document.title"
openclaw browser --browser-profile linear requests --filter api --json
```

Tips:

- Prefer snapshot refs over brittle CSS selectors.
- Capture both UI and network signals when validating status changes.

### Upload and download artifacts

```bash
# Upload a file into the focused page input.
openclaw browser --browser-profile linear upload ./artifacts/release-notes.pdf

# Trigger export/download from UI, then inspect browser downloads.
openclaw browser --browser-profile linear downloads
```

For scripted pipelines, use `--json` on `downloads` and parse file paths/statuses in your automation.

## Comet compatibility

Some hosted browser services or wrappers (including some Comet setups) may not expose full CDP endpoints.

### Detect CDP support

Check whether the endpoint responds to standard DevTools discovery routes:

```bash
curl -fsS "http://127.0.0.1:9222/json/version"
curl -fsS "http://127.0.0.1:9222/json/list"
```

If either call fails, times out, or returns non-CDP output, treat attach as unsupported for this endpoint.

### Fallback when attach fails

If remote attach fails for the Comet profile, switch that workflow to an OpenClaw-managed Chromium profile:

```json5
{
  browser: {
    defaultProfile: "comet-managed",
    profiles: {
      "comet-managed": { cdpPort: 18812, color: "#FF6A00" },
    },
  },
}
```

Then run automation with:

```bash
openclaw browser --browser-profile comet-managed start
```

This fallback keeps session persistence and deterministic automation even when external CDP attach is unavailable.

## See also

- [Browser](/tools/browser)
- [Browser login + X/Twitter posting](/tools/browser-login)
- [Browser troubleshooting](/tools/browser-linux-troubleshooting)
