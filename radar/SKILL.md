---
name: radar
description: "Operational CS workflow for CSMs, AEs, and leaders. Triggers: 'show my radar', 'what's renewing', 'expansion pipeline', 'what's on my plate', 'top accounts this week', 'activity gap', 'stale accounts', 'team compliance', 'who needs coaching', 'what should I escalate', 'team risk', 'heat map', 'portfolio view', 'segment story', 'QBR prep', 'ARR risk', 'update for [account]'. Individual: renewal triage, expansion pipeline, plate management, activity gaps, account updates. Leader (FLL+): compliance, heat-map, portfolio. Gainsight MCP required. Staircase optional. Do NOT use for briefs or 1:1 prep (Oracle), quick lookups (Ask Gainsight), or ARR analytics (ask-snowflake-analyst)."
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
