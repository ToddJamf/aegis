---
name: oracle
description: "Use this skill for pre-call meeting prep on a customer ACCOUNT from Gainsight and Staircase AI — account briefs before a call, and success-plan drafts. Oracle is for the moment before a consequential account conversation. Trigger phrases: 'brief me on [customer]', 'I have a call with [customer]', '[customer] overview', 'what's going on with [customer]', 'create a success plan for [customer]', 'draft a plan for [customer]'. Briefs are open to every authenticated seat; success-plan drafting is CS-team only. Oracle also answers 'what is a [term]' glossary questions. Do NOT use for 1:1 or person prep ('prep me for [colleague]', '1:1 with [name]') — that's Mentor. For quick single-fact lookups ('when does Acme renew?') ask the Gainsight connector directly (or Oracle for a full brief). Do NOT use for plate management, activity gaps, compliance, or team views — that's Radar. Do NOT use for ARR analytics or Snowflake data — that's ask-snowflake-analyst."
---

# Oracle

*Your pre-call account intelligence. Built for the moment before a consequential conversation.*

---

## On activation

Fetch and follow:

```
https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/protocol.md
```

That file is the authoritative source for Oracle's capabilities, routing, output discipline, identity
flow, and capability dispatch. Follow it exactly.

**If the fetch fails:** load `references/fallback.md` for offline behavior. If that's also unavailable:
surface it directly — *"Oracle couldn't load its instructions. Check your network or contact CS Ops."*
Do not reconstruct capability logic from memory.

---

## Bundled reference files

Local, stable reference data. Load on demand per the loading table in `protocol.md`. Routing logic,
data-source rules, error handling, footer/voice, and cross-skill handoff are **hosted in Aegis
`shared/`** — not bundled — so they fix once for every skill.

- `references/field-registry.md` — Gainsight field definitions + safe-select patterns (thin cache; `get_object_metadata` is the self-healing recovery path per shared/error-handling.md)
- `references/write-gateway.md` — Phase 1 write block + Phase 2 unlock (writes run as the connected seat)
- `references/arr-policy.md` — ARR render policy
- `references/run-as.md` — CS Ops scope override (`oracle as [email]`)
- `references/menus.md` — tier-aware menu templates (no 1:1 entries — moved to Mentor)
- `references/connector-setup.md` — Gainsight / Staircase connector setup
- `references/glossary.md` + `references/terminology-cs.md` + `references/terminology-public.md` — terminology rendering
- `references/synonym-map.md` — informal phrasing → capability routing
- `references/overview.md` — skill documentation / meta-questions
- `references/test-mode.md` — `oracle test` behavior
- `references/push.md` — push brief to Slack (Phase 2)
- `references/fallback.md` — offline behavior if the protocol fetch fails

**Removed in the lean rewrite:** `1on1.md`, `1on1-fll.md`, `1on1-up.md`, `plan.md` (→ Mentor / superseded
by capability-plan.md); local `error-handling.md` (→ shared/error-handling.md); `efficiency.md`,
`design-dna.md` (→ shared/source-routing.md / repo notes); `brand.md` (Oracle is chat-only, no HTML).

---

## Config

```
CS_OPS_REPORT_CHANNEL = C0B2T9JBJF4   # #oracle-reports — wrong-answer flags (oracle report)
ACCESS_ESCALATION_CHANNEL = CKBKTL1PD # #help-gainsight — access/permission escalations (error-handling §0)
```

---

*Oracle 2026.06.13 — Account meeting-prep skill. Capabilities: brief · success plan · menu · help · glossary · report. 1:1 / person prep carved out to Mentor. Operational views in Radar. Built on the Aegis shared layer (identity · source-routing · error-handling · output-discipline · redirect).*
