---
description: AtomClaw healthcheck — verify runtime, persona, repo hygiene, and workflow surface (safe, non-destructive)
---

# AtomClaw Healthcheck Workflow

**Goal:** Produce a fast, structured, low-risk audit of AtomClaw’s current state and recommend next actions.

**Scope:** Read-only checks by default. No destructive actions. No dependency changes. No auto-commits.  
**Safety:** Must respect `AGENTS.md` risk model and `TOOLS.md` constraints.

---

## When to use

- Atom is “acting weird”
- You’ve changed persona files and want to confirm runtime is reading them
- You’ve pulled changes / updated config
- Before adding new workflows or agents
- Before a longer SWE task

---

## Inputs (ask only if needed)

If any of these are unknown, ask Dane a single targeted question:

- **Repo path** for AtomClaw dev repo (default: `~/dev/ai-agents/AtomClaw`)
- **Workspace path** for live persona (default: `~/.openclaw/workspace-atomclaw`)
- **Target channel** being used (Telegram, etc.)

---

## Checklist (run in order)

### 1) Runtime Integrity (OpenClaw)

Run:

```bash
openclaw --version
openclaw agents list
openclaw config get bindings
openclaw channels status --probe || true
