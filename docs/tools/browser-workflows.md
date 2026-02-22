---
summary: "Per-service browser automation runbooks for Canva, Lovable.dev, and Comet"
read_when:
  - Automating Canva, Lovable.dev, or Comet with OpenClaw browser profiles
  - Setting up durable login sessions for SaaS automation
  - Handling Comet remote CDP compatibility and fallback
title: "Browser workflows for SaaS"
---

# Browser workflows for SaaS

This guide provides concrete runbooks for Canva, Lovable.dev, and Comet using dedicated OpenClaw browser profiles.

## Shared setup and safety baseline

Define one persistent profile per service under `browser.profiles.<name>` and keep names stable.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "canva",
    profiles: {
      canva: { cdpPort: 18830, color: "#00C4CC" },
      lovable: { cdpPort: 18831, color: "#7C3AED" },
      comet: { cdpUrl: "http://127.0.0.1:9222", color: "#FF6A00" },
      "comet-managed": { cdpPort: 18832, color: "#FF6A00" },
    },
  },
}
```

Session and credential handling rules:

- Perform first login manually for each profile so MFA and consent prompts are completed once.
- Reuse the same profile name for every later run to preserve cookies and local storage.
- Treat profile data as sensitive because authenticated sessions are stored there.
- Do not commit credentials, tokenized `cdpUrl` values, cookie exports, or screenshots containing secrets.
- Prefer environment variables or a secrets manager for any remote CDP credentials.

## Canva workflow

### Profile setup and auth persistence

Use a dedicated managed profile:

```json5
{
  browser: {
    profiles: {
      canva: { cdpPort: 18830, color: "#00C4CC" },
    },
  },
}
```

Bootstrap and persist login:

```bash
openclaw browser --browser-profile canva start
openclaw browser --browser-profile canva open "https://www.canva.com/login"
```

After manual login completes, restart and verify the session is still active:

```bash
openclaw browser --browser-profile canva stop
openclaw browser --browser-profile canva start
openclaw browser --browser-profile canva open "https://www.canva.com/"
openclaw browser --browser-profile canva snapshot --interactive
```

### Operation sequence

1. Open Canva home or workspace.
2. Navigate to the target design/project URL.
3. Upload local assets.
4. Export/download the output.
5. Verify completion through UI confirmation + download listing.

```bash
openclaw browser --browser-profile canva open "https://www.canva.com/"
openclaw browser --browser-profile canva navigate "https://www.canva.com/folder/all-your-designs"
openclaw browser --browser-profile canva upload ./artifacts/brand-kit/logo.png
openclaw browser --browser-profile canva snapshot --interactive
openclaw browser --browser-profile canva downloads --json
openclaw browser --browser-profile canva screenshot --full-page
```

## Lovable.dev workflow

### Profile setup and auth persistence

Use a dedicated managed profile:

```json5
{
  browser: {
    profiles: {
      lovable: { cdpPort: 18831, color: "#7C3AED" },
    },
  },
}
```

Bootstrap and persist login:

```bash
openclaw browser --browser-profile lovable start
openclaw browser --browser-profile lovable open "https://lovable.dev/login"
```

After manual login, validate persistence with a restart:

```bash
openclaw browser --browser-profile lovable stop
openclaw browser --browser-profile lovable start
openclaw browser --browser-profile lovable open "https://lovable.dev/projects"
openclaw browser --browser-profile lovable snapshot --interactive
```

### Operation sequence

1. Open Lovable.dev and go to project list.
2. Navigate to a specific project/editor.
3. Upload artifacts (brief, image, or supporting file).
4. Trigger build or export workflow.
5. Verify completion from status text and network calls.

```bash
openclaw browser --browser-profile lovable open "https://lovable.dev/projects"
openclaw browser --browser-profile lovable navigate "https://lovable.dev/projects/<project-id>"
openclaw browser --browser-profile lovable upload ./artifacts/spec.md
openclaw browser --browser-profile lovable snapshot --interactive --json
openclaw browser --browser-profile lovable requests --filter api --json
openclaw browser --browser-profile lovable downloads --json
```

## Comet workflow

### Profile setup and auth persistence

Use a remote CDP profile when Comet exposes compatible endpoints, plus a managed fallback profile.

```json5
{
  browser: {
    profiles: {
      comet: { cdpUrl: "http://127.0.0.1:9222", color: "#FF6A00" },
      "comet-managed": { cdpPort: 18832, color: "#FF6A00" },
    },
  },
}
```

For remote attach, keep credentials out of tracked files and prefer secret-backed env var interpolation in your runtime.

Bootstrap login on whichever profile you plan to run routinely (`comet` or `comet-managed`), then restart and verify the logged-in landing page with `snapshot --interactive`.

### Comet CDP compatibility checks

Run discovery checks before automation:

```bash
curl -fsS "http://127.0.0.1:9222/json/version"
curl -fsS "http://127.0.0.1:9222/json/list"
openclaw browser --browser-profile comet status
openclaw browser --browser-profile comet start
```

Treat attach as incompatible when any check fails, times out, returns invalid DevTools JSON, or repeatedly disconnects during startup.

### Fallback path to managed Chromium

When attach fails, switch to the managed profile and continue the workflow:

```bash
openclaw browser --browser-profile comet-managed start
openclaw browser --browser-profile comet-managed open "https://comet.example"
openclaw browser --browser-profile comet-managed snapshot --interactive
```

### Operation sequence

1. Open Comet app.
2. Navigate to target workspace/project.
3. Upload required artifacts.
4. Download generated output.
5. Verify completion in UI and request log.

```bash
openclaw browser --browser-profile comet open "https://comet.example"
openclaw browser --browser-profile comet navigate "https://comet.example/projects/<project-id>"
openclaw browser --browser-profile comet upload ./artifacts/input.json
openclaw browser --browser-profile comet requests --filter api --json
openclaw browser --browser-profile comet downloads --json
openclaw browser --browser-profile comet screenshot --full-page
```

## Troubleshooting

| Breakage       | Likely cause                                          | Check                                                                  | Fix                                                                                        |
| -------------- | ----------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Login expired  | Session timeout, revoked cookies, or SSO revalidation | `snapshot --interactive` on landing page shows login screen            | Re-auth on the same profile and re-run stop/start persistence check                        |
| Selector drift | UI layout or labels changed                           | Capture fresh snapshot refs and inspect current role tree              | Update automation steps to current refs, avoid stale hardcoded selectors                   |
| CDP disconnect | Remote Comet CDP endpoint unstable or incompatible    | Check `/json/version`, `/json/list`, and `browser status/start` output | Retry once, then switch to `comet-managed` profile for managed Chromium control            |
| Blocked upload | Hidden input, permission prompt, or invalid path      | Confirm local file exists and snapshot shows active upload UI          | Re-focus upload surface, run `upload` again, and verify through UI + downloads/API signals |

## See also

- [Browser](/tools/browser)
- [Personal SWE agent preset](/automation/personal-swe-agent)
- [Browser login + X or Twitter posting](/tools/browser-login)
- [Browser troubleshooting](/tools/browser-linux-troubleshooting)
