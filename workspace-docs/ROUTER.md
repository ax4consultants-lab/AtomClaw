# ROUTER.md – Model Routing & Tier Rules (v1.5 – current)

This file defines how thinking / generation / research tasks are routed between local models, lightweight engines, and heavy models.

Goal: keep costs sane, keep sensitive/compliance tasks locked down, make routing deterministic and auditable.

Last updated: 2025-xx-xx (update date when you change bindings)

---

## Current Tier Structure (v1.5)

- Tier 0 – Local / fast / cheap / private (default for almost everything)
- Tier 2 – Research / synthesis / external knowledge lookup
- Tier 3 – Compliance gate / publish-ready reasoning / final safety pass

No Tier 1 anymore.  
No Codex anywhere.  
GPT-4o is **not** default Tier 3.

---

## Tier Summary (Authoritative Defaults)

| Tier | Purpose                                                                               | Default Profile / Engine                      | When used                                                                       | Cost / Privacy level     |
| ---- | ------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------ |
| 0    | Default planning, building, patching, drafting                                        | **local-tier0 (default)**                     | PLAN, BUILD, PATCH, SEO draft, most internal reasoning, code, docs              | Very low / fully private |
| 2    | Research, comparisons, conflict resolution, intel gathering                           | **siliconflow-research**                      | Explicit RESEARCH or INTEL tasks only                                           | Low–medium / controlled  |
| 3    | Compliance pass, legal/sensitive tightening, pre-publish sanity, escalation reasoning | **openai-gpt5-mini (mandatory publish gate)** | Explicit compliance review / publish-ready finalisation / high-stakes reasoning | Medium–high / OpenAI     |

### Tier 0 (Draft Engine) Optionality (Controlled)

SiliconFlow draft models may be used as a Tier 0 “draft engine” only when:

- The task is explicitly SEO batch drafting (or high-volume drafting), AND
- A quick local-tier0 attempt is insufficient or would be materially slower, AND
- A cost note is included in PLAN when the batch is non-trivial.

Otherwise: Tier 0 stays on **local-tier0**.

---

## Hard Routing Rules (must never be broken)

| Task / Intent                 | Must route to | Must NOT route to | Notes / failure condition                                   |
| ----------------------------- | ------------- | ----------------- | ----------------------------------------------------------- |
| PLAN                          | Tier 0        | Tier 2 or Tier 3  | Any Tier 3 call on PLAN = routing violation                 |
| BUILD / PATCH / refactor      | Tier 0        | Tier 2 or Tier 3  | Same as above                                               |
| SEO draft / content outline   | Tier 0        | Tier 3            | Tier 3 only allowed after explicit compliance request       |
| RESEARCH / deep comparison    | Tier 2        | Tier 0 or Tier 3  | RESEARCH must never hit Tier 0 or jump straight to Tier 3   |
| INTEL / background gathering  | Tier 2        | Tier 0 or Tier 3  | Same                                                        |
| Compliance / legal tightening | Tier 3        | Tier 0 or Tier 2  | Must hit gpt5-mini for publish-ready outputs; no exceptions |
| Escalation reasoning          | Tier 3        | Tier 0 or Tier 2  | Only when explicitly requested or clearly high-stakes       |

---

## Tier 3 (gpt5-mini) – What it is actually for

- Rewrite / tighten language to remove risky claims
- Check for legal/compliance red flags
- Convert drafts into publish-ready copy
- Final sanity pass before anything goes public or to client
- High-stakes reasoning when explicitly requested

It is **not** for normal planning, coding, or casual research.

---

## Publish-Ready Artefact Rule (SEO + Public Content)

When Tier 3 is used for SEO or any publish-ready content:

- Must read:
  - `/drafts/{slug}.md`
- Must write:
  - `/drafts/{slug}-final.md`
- Must **not** overwrite the original draft.
- The `-final.md` file is the only artefact considered “publish-ready”.

If `-final.md` does not exist → publish-ready output does not exist.

---

## Quick Checklist – Before Any Routing Change

☐ Tier 3 is bound to **openai-gpt5-mini** (not gpt4o)  
☐ No codex profile exists anywhere  
☐ Tier 0 default profile is **local-tier0**  
☐ No automatic escalation from Tier 0 → Tier 3  
☐ RESEARCH / INTEL tasks never hit Tier 0 or Tier 3  
☐ PLAN / BUILD / PATCH always stay Tier 0  
☐ Compliance-style tasks (e.g. “make this legally safer”) trigger Tier 3  
☐ All routing decisions are logged with tier + reason + confidence + profile/model

---

## Test Commands to Verify Routing (run these when in doubt)

1. "Plan the next 3 steps for the invoice parser refactor"  
   → must stay Tier 0 (local-tier0)

2. "Research current best practices for secure webhook verification in 2025"  
   → must go Tier 2 (siliconflow-research)

3. "Rewrite this paragraph to remove any risky compliance claims and tighten legal language: [paste risky text]"  
   → must go Tier 3 (openai-gpt5-mini)

4. "SEO draft asbestos-register-adelaide"  
   → must be Tier 0 and write `/drafts/asbestos-register-adelaide.md`

5. "SEO finalize asbestos-register-adelaide"  
   → must be Tier 3 and write `/drafts/asbestos-register-adelaide-final.md`

If any of those go to the wrong tier → routing is broken → do not ship anything until fixed.

---

## Change Process

Any change to tiers, models, or routing rules must:

- Be committed here first
- Be tested with the commands above
- Be noted with date + reason + new bindings
- Be announced in main session ("Routing updated – see ROUTER.md")

Keep this file honest.  
If routing drifts, future me will break in fun but expensive ways.
