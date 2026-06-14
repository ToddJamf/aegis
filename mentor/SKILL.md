---
name: mentor
description: "Mentor is 1:1 and people prep for CS leaders — prepping for a conversation with a PERSON, not an account. Use for: 1:1 prep with a direct report (an FLL prepping for a CSM, a Segment Leader prepping for an FLL, and up the chain), and prep for a 1:1 with your own manager (any CS tier, upward). Trigger phrases: '1:1 with [name]', 'prep me for [name]', '1:1 prep [name]', 'prep for my 1:1 with [manager]'. 1:1 prep for a direct report is CS-leader (FLL+) only; upward prep with your own manager is any CS tier. Do NOT use for account briefs or success plans ('brief me on [customer]', 'draft a plan for [customer]') — that's Oracle. Do NOT use for operational team views (what's on my plate, compliance, heat-map, portfolio) — that's Radar. Do NOT use for quick factual lookups (the Gainsight connector directly). NOTE: pre-UAT — explicit invocation only until validated."
disable-model-invocation: true
---

# Mentor

*1:1 prep for the conversation about a person. Oracle preps you for the account; Mentor preps you for the human across the table.*

---

## On activation

Fetch and follow:

```
https://raw.githubusercontent.com/ToddJamf/aegis/main/mentor/protocol.md
```

That file is the authoritative source for Mentor's routing, identity flow, direction detection, and
capability dispatch. Follow it exactly.

**If the fetch fails:** load `references/fallback.md`. If that's also unavailable: surface it directly —
*"Mentor couldn't load its instructions. Check your network or contact CS Ops."* Do not reconstruct
capability logic from memory.

---

## Bundled reference files

Local, stable data. Routing, data-source rules, error handling, footer/voice, and cross-skill handoff
are hosted in Aegis `shared/` — not bundled.

- `references/field-registry.md` — Gainsight field definitions + safe-select patterns (thin cache; `get_object_metadata` is the self-healing recovery path per shared/error-handling.md)
- `references/fallback.md` — offline behavior if the protocol fetch fails

---

## Why `disable-model-invocation: true`

Mentor ships **pre-UAT** and must not auto-trigger by accident (Bluhm B8 pattern). Until it passes the
cold-run UAT (one FLL + one Segment Leader), it runs on **explicit invocation only** — "Mentor, 1:1 with
[name]". Oracle's redirect names Mentor and rephrases the ask, so users still land here on purpose.
Remove this line once UAT passes (charter step 8).

---

*Mentor 2026.06.13 — Person / 1:1 prep skill. Carved from Oracle (account vs. person boundary). Capabilities: 1:1 prep (direction DOWN to a report, direction UP to your manager). Built on the Aegis shared layer (identity · source-routing · error-handling · output-discipline · redirect). Pre-UAT: explicit invocation only.*
