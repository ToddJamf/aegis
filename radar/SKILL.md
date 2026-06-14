---
name: radar
description: "Operational CS workflow for CSMs, AEs, and leaders. Triggers: 'show my radar', 'what's renewing', 'expansion pipeline', 'what's on my plate', 'top accounts this week', 'activity gap', 'stale accounts', 'team compliance', 'who needs coaching', 'what should I escalate', 'team risk', 'heat map', 'portfolio view', 'segment story', 'QBR prep', 'ARR risk'. Individual: renewal triage, expansion pipeline, plate management, activity gaps. Leader (FLL+): compliance, heat-map, portfolio. Gainsight MCP required. Staircase optional. Do NOT use for briefs or 1:1 prep (Oracle), or ARR analytics (ask-snowflake-analyst). Quick single-fact lookups are served by the Gainsight connector directly."
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

Local, stable reference data. Load on demand per the loading table in `protocol.md`. Routing logic,
data-source rules, error handling, footer/voice, and cross-skill handoff are **hosted in Aegis
`shared/`** — not bundled — so they fix once for every skill (identity-leader-tiers · source-routing ·
error-handling · output-discipline · redirect).

- `references/identity.md` — CSM/AE persona detection (book field, Hybrid toggle; AE = `Primary_Territory_Owner__gc`). The local half of the dual identity model — leader-tier detection is `shared/identity.md`.
- `references/scoring.md` — R1–R5 renewal scores + E1–E4 expansion scores (full formulas + edge cases)
- `references/artifact.md` — Full HTML artifact template (two-tab renewal/expansion render)
- `references/field-registry.md` — Gainsight field definitions + safe-select patterns (`get_object_metadata` is the self-healing recovery path per shared/error-handling.md §A1)
- `references/arr-policy.md` — ARR render policy (scope-aware: full $ for owned/leader, ARR_Band for out-of-scope list caps)
- `references/connector-setup.md` — Connector setup instructions (Gainsight / Staircase)
- `references/fallback.md` — offline behavior if the protocol fetch fails

---

*Radar 2026.06.13 — Migrated onto the Aegis shared layer. Output discipline + footer (lean variant) delegated to shared/output-discipline.md; cross-skill boundaries to shared/redirect.md; no-access to error-handling §0. Dual identity model kept: references/identity.md (CSM/AE persona) + shared/identity.md (leader tiers). Enhancements layered in: unified Staircase fan-out (B1, plate), risk×expansion recency (B2, expansion/renewal), cleanup-before-create gate (B6, plate/gaps), report mining (leader views). R/E scoring and tech-touch logic unchanged.*
*2026-06-03 (v2.0) — Expanded from renewal/expansion-only (v1.3) to full operational CS workflow: my-plate, my-gaps, compliance, heat-map, portfolio, update.*
