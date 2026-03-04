# MODEL_ROUTER.md

AtomClaw v1.5 — Router + Lane Enforcement Spec (Codex removed, $400/mo governed)

## 1) Purpose

ModelRouter is the single entry point that:

- Parses command grammar
- Selects lane (Tier 0 / Tier 2 / Tier 3)
- Selects provider/profile (auth-profiles)
- Enforces approval gates + budget discipline
- Logs every decision (lane, profile, model, token estimate)

No direct model calls outside ModelRouter.

## 2) Lanes (v1.5)

### Tier 0 — Default / Local Work

Drafting, audits, repo discovery, file transforms, code edits (including multi-file, but approval-gated).
No external browsing.
Primary profiles: local-tier0 (optionally siliconflow-draft if you allow it as Tier 0 “draft engine”).

### Tier 2 — Research Only (Explicit)

External intel only (SERPs, competitors, compliance updates, public vulns).
Invoked only by: RESEARCH:, INTEL:, or SEO: research …
Must return summarised intel to Tier 0.
Never triggers BUILD automatically.

### Tier 3 — Compliance / Escalation

Used for:

- Mandatory publish-ready compliance pass (GPT-5 mini)
- Rare escalation reasoning (explicit)
  Never default.

## 3) Command Grammar → Lane Mapping (Hard Rules)

The first token determines lane:

### Tier 0 commands

PLAN:
BUILD:
PATCH:
REVIEW:
RUN:
DRYRUN:
HEALTH:
MEMORY:
REPO:
VOICE:
SECSCAN:
SEO: (except SEO: research)

### Tier 2 commands (external)

RESEARCH:
INTEL:
SCRAPE:
SEO: research

### Tier 3 commands

COMPLIANCE: (optional alias; see section 5)
ESCALATE: (optional alias; explicit only)

Ambiguous input defaults to PLAN: (Tier 0).

## 4) Router Decision Tree (Deterministic)

1. Parse command prefix.
2. Assign lane based on mapping above.
3. Apply gates:

- If lane is Tier 2 → require Plan Gate if the output is intended to drive build actions.
- If output is “publish-ready” → require Tier 3 compliance pass step.

4. Pick profile:

- Tier 0: local-tier0 (default)
- Tier 2: siliconflow-research (default)
- Tier 3: openai-gpt5-mini (mandatory for publish gate)

5. Execute with rate-limit/backpressure rules.
6. Log:

- command, lane, profile, model, token estimate, cost estimate, artefact paths

## 5) Publish-Ready Compliance Gate (Mandatory)

Definition: any content intended for public/client consumption:

- Ax4 service pages
- blog posts
- compliance-facing templates
- client emails (when high-risk)

Rule:

- Tier 0 produces `/drafts/{slug}.md`
- Tier 3 (GPT-5 mini) produces `/drafts/{slug}-final.md`
  No publish-ready output exists until `-final.md` exists.

Recommended explicit command:

- COMPLIANCE: seo_finalize {slug}
  OR
- SEO: finalize {slug}

Router must force Tier 3 profile for finalize.

## 6) SEO Skill Routing (v1.5)

### SEO: draft {slug}

- Tier 0
- Uses local-tier0 by default
- Writes:
- /drafts/{slug}.md
- /analysis/{slug}-notes.md (optional but recommended)

### SEO: audit {slug}

- Tier 0
- Uses local-tier0
- Writes:
- /analysis/{slug}-audit.md

### SEO: research {query}

- Tier 2
- Uses siliconflow-research
- Writes:
- /analysis/seo-research-{date}.md
- Must include sources

### SEO: finalize {slug}

- Tier 3 (mandatory)
- Uses openai-gpt5-mini
- Writes:
- /drafts/{slug}-final.md

## 7) Budget Governance Hooks ($400/mo cap)

Router must maintain counters (rolling monthly):

- Tier 2 spend estimate
- Tier 3 spend estimate (GPT-5 mini passes)
  Hard behaviour:
- If budget risk detected → Router requires PLAN confirmation before executing Tier 2 or Tier 3 batches.
- Prefer batching compliance passes (e.g., finalize 2 pages at once) only if it reduces overhead and remains reviewable.

## 8) Prohibited Behaviour

Router must block:

- Tier 2 invoked without explicit command prefix
- Tier 3 invoked without explicit finalize/escalate intent (except mandatory publish gate)
- Any “auto-build” from Tier 2 outputs
- Any publish action (WP/API) unless Gate C enabled

## 9) Logging Requirements

Each run must log:

- timestamp
- command
- lane selected
- profile selected
- model selected
- artefacts written
- token estimate + actual (if available)
- errors + backoff events
