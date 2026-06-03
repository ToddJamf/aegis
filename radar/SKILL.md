---
name: radar
description: >
  Use this skill for all operational CS workflow — both individual and leader views.
  Individual (CSM/AE): renewal triage, expansion pipeline, plate management, activity gaps, account updates.
  Leader (FLL+): team compliance, heat-map / escalations, portfolio / segment story.
  Trigger phrases include: "show my radar", "pull up my radar", "what's renewing", "expansion pipeline",
  "what's on my plate", "top accounts this week", "what should I work on", "activity gap", "stale accounts",
  "team compliance", "who needs coaching", "what should I escalate", "team risk", "heat map",
  "portfolio view", "segment story", "QBR prep", "ARR risk", "renewal exposure",
  "update for [account]", "draft an account update".
  Works for CSMs (Csm field) and AEs (Primary_Territory_Owner__gc).
  Gainsight MCP required. Staircase MCP optional — degrades gracefully without it.
  Do NOT use for pre-call briefs, 1:1 prep, or success plan drafts — use Oracle.
  Do NOT use for quick factual lookups — use Ask Gainsight.
  Do NOT use for ARR analytics or Snowflake data — use ask-snowflake-analyst.
---

# Radar

*Operational CS workflow. Renewal triage, expansion pipeline, plate management, and team health — for CSMs, AEs, and CS leaders.*

---

## On activation

Fetch and follow:

```
https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/protocol.md
```

That file is the authoritative source of truth for Radar's capabilities, routing logic, output discipline, identity flow, and all capability dispatch. Follow its instructions exactly.

**If the fetch fails:** Load `references/fallback.md` for offline behavior. If unavailable: *"Radar couldn't load its instructions. Check your network connection or contact CS Ops."* Do not reconstruct capability logic from memory.

---

## Bundled reference files

Load on demand per the reference loading table in `protocol.md`:

- `references/identity.md` — CSM/AE persona detection (book field, hybrid toggle)
- `references/scoring.md` — R1–R5 renewal scores + E1–E4 expansion scores (full formulas + edge cases)
- `references/artifact.md` — Full HTML artifact template (two-tab renewal/expansion render)
- `references/field-registry.md` — Gainsight field definitions and safe-select patterns
- `references/arr-policy.md` — ARR render policy (ownership check for list views)
- `references/update.md` — Account update spec (Spotlight bi-weekly / Grow/Retain/React monthly)
- `references/connector-setup.md` — Connector setup instructions (Gainsight / Staircase)

---

*Radar v2.0 (2026-06-03) — Expanded from renewal/expansion-only (v1.3) to full operational CS workflow skill. Adds: my-plate, my-gaps, team compliance, heat-map, portfolio, update. Renewal/expansion capabilities unchanged. Built on Aegis hosted logic framework.*
