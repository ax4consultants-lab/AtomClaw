# skill: seo

# purpose: Dispatch SEO commands with tier discipline + artefacts + logs.

# inputs: user_message

# outputs: artefacts in /drafts and /analysis, log lines in /logs/seo-dispatcher.log

## Rules

- Only accept commands:
  - SEO: draft <slug>
  - SEO: research <query>
  - SEO: finalize <slug>
- Hard fail on any other syntax.
- Enforce tier requirements:
  - draft → Tier 0 (local-tier0)
  - research → Tier 2 (siliconflow-deepseek)
  - finalize → Tier 3 (openai-gpt5-mini)
- No fallback:
  - Tier 2/3 must fail if profile mismatch/unavailable.
  - Must not silently run on Tier 0.
- Artefacts:
  - draft writes /drafts/{slug}.md
  - research writes /analysis/seo-research-{YYYYMMDD}.md with sources
  - finalize reads /drafts/{slug}.md and writes /drafts/{slug}-final.md
- Log every run to /logs/seo-dispatcher.log as JSON line with:
  timestamp, command, tier, profileId, model, status, reason, artefacts

## Execution (Operator)

If OpenClaw cannot select profiles per call:

- The dispatcher must CHECK current model/profile context.
- If mismatch: print ERROR_PROFILE_MISMATCH and STOP.
- It must tell the operator which profile is required and how to rerun.

## Workflow Links

Draft workflow: skills/seo/ax4_seo_task.md
Research workflow: skills/seo/service_campaign_v1.md
Finalize workflow: skills/seo/asbestos_register_upgrade_v1.md
Dispatcher spec: skills/seo/dispatcher.md
