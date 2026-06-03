---
name: oracle
description: "Use this skill whenever anyone needs meeting prep from Gainsight or Staircase AI — pre-call account briefs, 1:1 prep for a CSM (FLL), 1:1 prep for an FLL (Segment Leader), 1:1 prep with your own manager (any tier), or success plan drafts. Oracle is for the moment before a consequential conversation. Trigger phrases include: 'brief me on [customer]', 'I have a call with [customer]', '[customer] overview', '1:1 with [name]', 'prep me for [name]', 'prep for my 1:1 with [name]', 'create a success plan for [customer]', 'draft a plan for [customer]'. All users can pull a brief or glossary lookup. 1:1 prep and success plans are CS-tier only. Do NOT use for quick factual lookups ('when does Acme renew?') — use Ask Gainsight instead. Do NOT use for plate management, activity gaps, compliance, or team heat-maps — use Radar instead. Do NOT use for ARR analytics or Snowflake data — use ask-snowflake-analyst."
---

# Oracle

*Your pre-call intelligence brief. Built for the moment before a consequential conversation.*

---

## On activation

Fetch and follow:

```
https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/protocol.md
```

That file is the authoritative source of truth for Oracle's capabilities, routing logic, output discipline, identity flow, and all capability dispatch. Follow its instructions exactly.

**If the fetch fails:** Load `references/fallback.md` for offline behavior. If `references/fallback.md` is also unavailable: surface the error directly — *"Oracle couldn't load its instructions. Check your network connection or contact CS Ops."* Do not attempt to reconstruct capability logic from memory.

---

## Bundled reference files

The following files are in this skill package. Load on demand per the reference loading table in `protocol.md`:

- `references/field-registry.md` — Gainsight field definitions and safe-select patterns
- `references/write-gateway.md` — Phase 1 write block + Phase 2 unlock path
- `references/arr-policy.md` — ARR render policy (unrestricted for brief/lookup; ownership check for lists)
- `references/run-as.md` — CS Ops scope override (`oracle as [email]`)
- `references/menus.md` — Tier-aware menu templates
- `references/overview.md` — Skill documentation and meta-questions
- `references/test-mode.md` — Test mode behavior (`oracle test`)
- `references/design-dna.md` — Architecture reference + build gate (8-principle checklist)
- `references/connector-setup.md` — Connector setup instructions (Gainsight / Staircase)
- `references/brand.md` — Visual brand style (HTML outputs)
- `references/push.md` — Push brief to Slack (Phase 2)
- `references/1on1.md` — 1:1 prep supplementary spec (DOWN / CSM target)
- `references/1on1-fll.md` — 1:1 prep supplementary spec (DOWN / FLL target)
- `references/1on1-up.md` — 1:1 prep supplementary spec (UP)
- `references/plan.md` — Success plan fallback spec
- `references/synonym-map.md` — Informal phrasing → capability routing map
- `references/efficiency.md` — Query payload design patterns
- `references/glossary.md` — Terminology rendering
- `references/terminology-cs.md` — Gainsight + CS terminology (CS tiers)
- `references/terminology-public.md` — Gainsight + CS terminology (Not-CS users)
- `references/error-handling.md` — Error patterns and recovery (local supplement to shared)

---

## Config

```
CS_OPS_REPORT_CHANNEL = C0B2T9JBJF4   # #oracle-reports — update here if channel changes
```

---

*Oracle v1.0 (2026-06-03) — Meeting prep skill. Carved from Gainsight Sentinel v3.7. Capabilities: brief · 1:1 (all directions) · success plan · menu · help · glossary. Operational capabilities moved to Radar. Quick lookup moved to Ask Gainsight. Built on Aegis hosted logic framework.*
