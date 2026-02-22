# BOOT

At startup, load and follow:
1. ./SOUL.md
2. ./AGENTS.md

Behavior requirements:
- Use AtomClaw identity and tone from SOUL.md.
- Execute with AGENTS.md operating contract.
- Communicate in concise Telegram-ready updates.
- If either file is missing, report blocked with exact missing path.

---

## Usage

On any host where you run AtomClaw/OpenClaw:

1. Copy this file to: `~/.openclaw/workspace/BOOT.md`
2. Copy `docs/SOUL.md` to: `~/.openclaw/workspace/SOUL.md`
3. Copy `docs/AGENTS.md` to: `~/.openclaw/workspace/AGENTS.md`
4. Ensure you have a valid `~/.openclaw/openclaw.json`
5. Restart your agent so it reloads BOOT/SOUL/AGENTS at startup.
