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
      systemPrompt: `
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
      `,
      hooks: {
        beforeCommand: {
          shell: [
            {
              name: "workspace-allowlist",
              match: "*",
              denyIfCwdOutside: ["~/code", "~/scratch/review"],
              message: "Command blocked. Run only inside approved repo roots.",
            },
            {
              name: "block-destructive-shell",
              matchRegex: "(^|\\s)(rm\\s+-rf|sudo\\s+rm|git\\s+clean\\s+-fdx)(\\s|$)",
              action: "deny",
              message: "Destructive shell command blocked by preset policy.",
            },
            {
              name: "require-checks-before-commit",
              matchRegex: "(^|\\s)git\\s+commit(\\s|$)",
              requireRecentCommands: ["pnpm test", "pnpm build"],
              within: "30m",
              action: "deny",
              message: "Run pnpm test and pnpm build successfully before commit.",
            },
          ],
          git: [
            {
              name: "branch-name-policy",
              on: "branch-create",
              requireNameRegex: "^(feat|fix|chore|docs)\\/[a-z0-9._-]+$",
              action: "deny",
              message: "Branch must match feat|fix|chore|docs/<topic>.",
            },
            {
              name: "commit-subject-policy",
              on: "commit-msg",
              requireSubjectRegex: "^(feat|fix|chore|docs|refactor|test)(\\([a-z0-9._-]+\\))?: .+",
              action: "deny",
              message: "Commit subject must follow Conventional Commit style.",
            },
            {
              name: "block-history-rewrite",
              on: "push",
              denyArgs: ["--force", "--force-with-lease"],
              action: "deny",
              message: "Force push is disabled in this preset.",
            },
          ],
          network: [
            {
              name: "allow-docs-and-metadata-only",
              action: "allowlist",
              hosts: ["docs.openclaw.ai", "registry.npmjs.org", "api.github.com", "github.com"],
              message: "Network access restricted to approved documentation and metadata hosts.",
            },
          ],
        },
      },
      scratchpad: {
        commitChecklist: [
          "Run pnpm test",
          "Run pnpm build",
          "Summarize changed files",
          "Write a Conventional Commit message",
        ],
        pullRequestTemplate: {
          requiredFields: ["Summary", "Changes", "Tests", "Risks", "Rollback", "Follow ups"],
        },
      },
    },
  },

  cron: {
    enabled: true,
    maxConcurrentRuns: 1,
    jobs: [
      {
        id: "maintenance-deps-audit",
        schedule: { kind: "cron", expr: "0 9 * * 1", tz: "UTC" },
        sessionTarget: "isolated",
        payload: {
          kind: "agentTurn",
          message: "Run dependency and lockfile health checks in approved repos and report only actionable issues.",
        },
        delivery: { mode: "announce", channel: "telegram", to: "tg:my-self-chat" },
      },
      {
        id: "maintenance-doc-link-check",
        schedule: { kind: "cron", expr: "0 15 * * 3", tz: "UTC" },
        sessionTarget: "isolated",
        payload: {
          kind: "agentTurn",
          message: "Run docs link checks and summarize broken links with exact file paths.",
        },
        delivery: { mode: "announce", channel: "telegram", to: "tg:my-self-chat" },
      },
      {
        id: "maintenance-pr-hygiene",
        schedule: { kind: "cron", expr: "0 12 * * 5", tz: "UTC" },
        sessionTarget: "isolated",
        payload: {
          kind: "agentTurn",
          message: "Review open branches for stale work and prepare next-step PR drafts with required sections.",
        },
        delivery: { mode: "announce", channel: "telegram", to: "tg:my-self-chat" },
      },
    ],
  },
}
```

## Policy examples included in the preset

### Allowed repo roots and workspaces

- Default workspace set with `agents.defaults.workspace: "~/code"`.
- Explicit guardrail allowlist keeps execution in `~/code` and `~/scratch/review`.

### Preferred branch naming

- Required branch name format:
  - `feat/<topic>`
  - `fix/<topic>`
  - `chore/<topic>`
  - `docs/<topic>`

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
