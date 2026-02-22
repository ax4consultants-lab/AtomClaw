---
summary: "Personal autonomous engineering preset with workspace, policy guardrails, cron loops, and PR workflow conventions"
read_when:
  - Building a personal software engineering automation profile
  - Restricting agent shell/git/network behavior for local repos
  - Standardizing branch, commit, test, and PR policies
  - Defining failure handling and Telegram status updates
title: "Personal SWE agent preset"
---

# Personal SWE agent preset

This guide provides a copy paste config preset for an autonomous personal software engineering workflow.

Use it when you want one agent profile that can:

- stay inside approved repo roots
- follow strict shell, git, and network policies
- run maintenance loops on a schedule
- enforce branch, commit, test, and PR conventions
- stop safely with clear Telegram status reports when blocked

## Complete preset

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      workspace: "~/code",
    },
  },
  cron: {
    enabled: true,
    maxConcurrentRuns: 1,
  },
}
```

> **Note:** The fields above are the only keys validated by `AgentDefaultsSchema` and the `cron` schema.
> Policy rules (system prompt, hooks, scratchpad) and cron job definitions are configured separately
> as described in the sections below.

## Implementation steps (commands and config locations)

Use this section to apply the preset with concrete CLI commands and file paths.

### 1) Set base workspace and cron config

The minimal JSON5 block above stays schema-valid in `~/.openclaw/openclaw.json`. Apply the same
values with CLI writes:

```bash
openclaw config set agents.defaults.workspace "~/code"
openclaw config set cron.enabled true
openclaw config set cron.maxConcurrentRuns 1
```

### 2) Add policy hooks with the supported hook layout

Per [Hooks](/automation/hooks), define each hook as a directory with `HOOK.md` and `handler.ts`.
Create policy hooks in your default workspace so they apply to this agent profile:

```bash
mkdir -p ~/code/hooks/workspace-allowlist
mkdir -p ~/code/hooks/block-destructive-shell
mkdir -p ~/code/hooks/require-checks-before-commit
mkdir -p ~/code/hooks/branch-name-policy
mkdir -p ~/code/hooks/commit-subject-policy
mkdir -p ~/code/hooks/block-history-rewrite
mkdir -p ~/code/hooks/allow-docs-and-metadata-only
```

Create one hook manifest per directory (same schema for each hook):

```yaml
---
name: workspace-allowlist
description: "Deny shell commands outside approved roots"
metadata: { "openclaw": { "emoji": "🛡️", "events": ["command"], "requires": { "bins": ["node"] } } }
---
```

Then implement each `handler.ts` with a default `HookHandler` export that enforces its policy
(workspace allowlist, destructive command blocking, commit/test gates, branch/commit regex checks,
and outbound domain allowlist).

After files are in place, verify discovery and enable hooks:

```bash
openclaw hooks list --verbose
openclaw hooks enable workspace-allowlist
openclaw hooks enable block-destructive-shell
openclaw hooks enable require-checks-before-commit
openclaw hooks enable branch-name-policy
openclaw hooks enable commit-subject-policy
openclaw hooks enable block-history-rewrite
openclaw hooks enable allow-docs-and-metadata-only
openclaw hooks check
```

### 3) Create maintenance cron jobs with `openclaw cron add`

Create concrete jobs matching the schedule in this preset:

```bash
openclaw cron add \
  --name "maintenance-deps-audit" \
  --cron "0 9 * * 1" \
  --tz "UTC" \
  --session isolated \
  --message "Run dependency and lockfile health checks for the current workspace, then summarize required updates."

openclaw cron add \
  --name "maintenance-doc-link-check" \
  --cron "0 15 * * 3" \
  --tz "UTC" \
  --session isolated \
  --message "Run docs link checks and report broken links with file paths and suggested fixes."

openclaw cron add \
  --name "maintenance-pr-hygiene" \
  --cron "0 12 * * 5" \
  --tz "UTC" \
  --session isolated \
  --message "Review stale branches and open pull requests, then prepare a concise hygiene summary and next actions."
```

### 4) Final verification checklist

Run this checklist to confirm the profile is active:

```bash
openclaw config get agents.defaults.workspace
openclaw config get cron.enabled
openclaw config get cron.maxConcurrentRuns
openclaw cron list
openclaw hooks check
openclaw channels status --probe
```

## Operations runbook (always-on validation)

Use this runbook to validate that your personal SWE agent stays healthy end-to-end across
startup, scheduled loops, and recovery events.

### Startup checklist

Run these commands in order when booting a machine or after onboarding changes:

```bash
openclaw gateway install
openclaw gateway status
openclaw status
openclaw channels status --probe
```

Confirm:

- daemon/service is installed and reports running
- gateway endpoint is reachable
- Telegram channel probe reports healthy auth and delivery path

### Loop checklist

Run this checklist to validate cron loop behavior and run visibility:

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
```

Confirm:

- cron scheduler is enabled and next wake is scheduled
- all expected jobs are registered and enabled
- last run status is visible (`ok`, `retry`, `blocked`, or explicit skip reason)
- retry and blocked outcomes are surfaced in run output for operator follow-up

### Recovery checklist

Use this sequence after crashes, host restarts, or channel reconnect incidents.

```bash
openclaw gateway restart
openclaw gateway status
openclaw logs --follow
openclaw cron status
openclaw channels status --probe
```

Reference paths for logs and state while recovering:

- gateway logs stream via `openclaw logs --follow`
- local config and state live under `~/.openclaw/`

Confirm after restart:

- gateway reports healthy status
- cron scheduler is enabled
- Telegram probe is healthy again

### Daily sanity checks command bundle

Run this bundle once per day to quickly verify status, cron, hooks, and channel health:

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw hooks check
openclaw channels status --probe
```

## Agent system prompt (conceptual policy)

Set your agent's behavioral policy by providing a system prompt through the gateway's agent
configuration interface. The following is an example policy you can adapt:

```text
You are my personal SWE agent for local repositories.

Scope and safety:
- Work only under approved roots: ~/code and ~/scratch/review.
- Never cd, read, or write outside approved roots.
- Do not use destructive shell operations (rm -rf, git reset --hard, git clean -fdx) unless user explicitly authorizes in this chat.

Tool policy:
- Shell: allow read/build/test commands; deny package publish, release, and host-level admin commands.
- Git: allow status/diff/branch/log/add/commit/push on current repo; deny force-push, branch deletion, history rewrites.
- Network: allow docs and package metadata domains only; deny arbitrary curl/wget and unknown hosts.

Branch and PR workflow:
- Branch naming must be feat/<topic>, fix/<topic>, chore/<topic>, or docs/<topic>.
- Commit subject must match: <type>(<scope>): <summary>.
- Before each commit, run required checks and capture pass/fail in notes.
- PR body must include: Summary, Changes, Tests, Risks, Rollback, Follow ups.

Failure handling:
- Retry budget: at most 2 retries per failing command, with one short diagnosis step between retries.
- Stop and ask user when blocked by missing credentials, ambiguous product decisions, destructive changes, or repeated test failures.
- Telegram status must include: objective, attempted commands, first actionable error, files changed, and next request for user.
```

## Hooks (conceptual policy)

The following hook definitions describe the intended enforcement policy for this preset.
Refer to the [Hooks](/automation/hooks) documentation for the supported configuration format.

**Shell hooks:**

- `workspace-allowlist`: deny commands run outside `~/code` or `~/scratch/review`.
- `block-destructive-shell`: deny `rm -rf`, `sudo rm`, and `git clean -fdx`.
- `require-checks-before-commit`: deny `git commit` unless `pnpm test` and `pnpm build` succeeded within the last 30 minutes.

**Git hooks:**

- `branch-name-policy`: deny branch creation unless the name matches `^(feat|fix|chore|docs)/[a-z0-9._-]+$`.
- `commit-subject-policy`: deny commit unless the subject matches Conventional Commit style.
- `block-history-rewrite`: deny `git push --force` and `git push --force-with-lease`.

**Network hooks:**

- `allow-docs-and-metadata-only`: restrict outbound requests to `docs.openclaw.ai`, `registry.npmjs.org`, `api.github.com`, and `github.com`.

## Scratchpad checklist (conceptual)

Use a scratchpad or notes section to track these items before each commit:

- Run `pnpm test`
- Run `pnpm build`
- Summarize changed files
- Write a Conventional Commit message

For PRs, ensure the body includes: Summary, Changes, Tests, Risks, Rollback, and Follow ups.

## Runnable GitHub workflow (CLI)

Use this sequence when the autonomous run needs to make a normal docs or code contribution
through GitHub.

### Prerequisites

Before changing files, confirm both GitHub auth and repository access:

1. GitHub CLI is installed: `gh --version`
2. CLI session/token is valid for the expected account: `gh auth status`
3. API access works with current token/session: `gh api user --jq .login`
4. Remote is reachable and you can read refs:
   - `git remote -v`
   - `git ls-remote --heads origin`
5. Push permission check (non-destructive probe):
   - `gh repo view --json nameWithOwner,viewerPermission --jq '.nameWithOwner + " " + .viewerPermission'`

If any prerequisite fails, stop and send a blocked Telegram update before attempting edits.

### Step-by-step command flow

Run the following commands in order, replacing placeholder values with your target repo and branch:

```bash
# 1) Clone or enter repo
git clone git@github.com:<owner>/<repo>.git
cd <repo>

# 2) Sync with default branch
git fetch origin
git checkout main
git pull --rebase origin main

# 3) Create a policy-compliant branch
git checkout -b docs/<topic>

# 4) Edit files
$EDITOR docs/automation/personal-swe-agent.md

# 5) Validate changes (adjust commands to repo standards)
pnpm test
pnpm build

# 6) Commit
git add docs/automation/personal-swe-agent.md
git commit -m "docs(automation): add runnable GitHub workflow guidance"

# 7) Push branch
git push -u origin docs/<topic>
```

### Pull request creation method (required)

Create the PR with GitHub CLI using a real multiline body (heredoc), not escaped `\n` strings.
The body must include all required sections: **Summary, Changes, Tests, Risks, Rollback, Follow ups**.

```bash
gh pr create \
  --base main \
  --head docs/<topic> \
  --title "docs(automation): add runnable GitHub workflow guidance" \
  -F - <<'EOF'
## Summary
- Brief objective and user impact.

## Changes
- Key file and behavior/documentation updates.

## Tests
- Commands run and pass/fail results.

## Risks
- Known risks, assumptions, or edge cases.

## Rollback
- Exact revert path (for example: revert commit or close PR without merge).

## Follow ups
- Any deferred cleanup or tracking items.
EOF
```

After creation, record the PR URL from `gh pr view --json url --jq .url`.

### Stop conditions and user-confirmation boundaries

Stop immediately and request explicit user confirmation before continuing when:

- auth/session checks fail (`gh auth status`, `gh api user`)
- repo permission is below write/maintain/admin
- push is rejected by branch protection or permission errors
- `git pull --rebase` has conflicts that require semantic decisions
- commit would include unrelated files or secrets
- required checks fail repeatedly and root cause is not clearly transient
- any destructive action is needed (`git push --force`, branch deletion, history rewrite)

Do **not** bypass these boundaries autonomously. Ask for a clear yes/no decision from the user.

### Blocked state Telegram update template (GitHub failures)

Use this template when blocked specifically by GitHub auth/push/CI issues:

```text
Status: blocked on GitHub workflow
Objective: <goal>
Step: <auth check | push | CI>
Commands: <exact commands attempted>
Failure type: <auth denied | push rejected | CI failed>
First actionable error: <single actionable line>
Changed files: <comma-separated list>
Retries: <n>/2
Need from you: <credential refresh | permission grant | merge/rebase guidance | CI override decision>
```

Failure-specific examples:

- **Auth denied**: request refreshed `gh auth login` session or token scope update.
- **Push rejected**: request branch protection guidance or permission grant.
- **CI fail**: request decision to fix forward, rerun, or pause merge.

## Cron maintenance jobs (conceptual)

The `cron` config key enables scheduled agent runs. The following jobs illustrate the intended
maintenance schedule. Refer to [Cron jobs](/automation/cron-jobs) for supported job configuration.

| Job                          | Schedule                  | Task                                     |
| ---------------------------- | ------------------------- | ---------------------------------------- |
| `maintenance-deps-audit`     | Every Monday 09:00 UTC    | Dependency and lockfile health checks    |
| `maintenance-doc-link-check` | Every Wednesday 15:00 UTC | Docs link checks, report broken links    |
| `maintenance-pr-hygiene`     | Every Friday 12:00 UTC    | Review stale branches, prepare PR drafts |

## Policy examples included in the preset

### Allowed repo roots and workspaces

- Default workspace set with `agents.defaults.workspace: "~/code"`.
- Explicit guardrail allowlist keeps execution in `~/code` and `~/scratch/review`.

### Preferred branch naming

- Required branch name format:
  - `feat/`
  - `fix/`
  - `chore/`
  - `docs/`

### Commit message pattern

- Required commit subject regex:
  - `^(feat|fix|chore|docs|refactor|test)(\([a-z0-9._-]+\))?: .+`

### Mandatory test and build before commit

- Commit is denied unless these commands succeeded recently:
  - `pnpm test`
  - `pnpm build`

### PR body template fields

- Required PR sections:
  - Summary
  - Changes
  - Tests
  - Risks
  - Rollback
  - Follow ups

## Failure handling for autonomous runs

Use these rules to keep autonomous work predictable and safe.

### Retry budget

- Maximum retries per failing command: **2**.
- Between retries, perform one focused diagnosis step:
  - inspect the first actionable error line
  - verify dependencies and current branch state
  - retry only if the root cause appears transient

### When to stop and ask user

Stop autonomous execution and ask for user input when any of these happen:

- missing credentials or unavailable secrets
- unclear product behavior or conflicting requirements
- destructive operation is required to continue
- same test or build failure persists after retry budget is exhausted
- policy denies a command needed for next progress

### What to include in Telegram status replies

For blocked or paused runs, send a concise status that includes:

1. objective and current step
2. exact commands attempted
3. first actionable error message
4. files changed so far
5. what was retried and retry count used
6. specific user decision or credential needed

Example status format:

```text
Status: blocked on test failure
Objective: add config preset docs for personal SWE agent
Commands: pnpm test; pnpm build
Error: vitest failed in docs link parser with missing fixture
Changed files: docs/automation/personal-swe-agent.md, docs/gateway/configuration.md
Retries: 2/2 used (same failure)
Need from you: confirm whether to update fixture baseline or skip this test in docs-only PR
```

## Related docs

- [Configuration](/gateway/configuration)
- [Configuration Reference](/gateway/configuration-reference)
- [Cron jobs](/automation/cron-jobs)
- [Hooks](/automation/hooks)
- [Browser workflows for SaaS (Canva, Lovable.dev, Comet)](/tools/browser-workflows)
