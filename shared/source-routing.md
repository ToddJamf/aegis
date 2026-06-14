# Shared — Source Routing (read source-of-truth)
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/source-routing.md
#
# Shared across: Oracle, Mentor, Radar, Pulse
# Answers ONE question: for a given data read, which tool / which source is authoritative?
# Skills do not decide this locally — they route here. Fix a routing rule once, every skill picks it up.
# Output rules: shared/output-discipline.md. Error recovery: shared/error-handling.md.

---

## The line — what's shared vs what stays in the skill

**Shared (this file): source routing.** Where data comes from. "Health = `ask_scorecard`", "timeline = semantic `fetch_timeline_activity_list`", "cross-account Staircase flags = synced Gainsight fields". Same answer no matter which skill asks. If two skills decide this independently, they drift — that's the bug the shared layer exists to kill.

**Per-skill: capability routing.** Which user intent maps to which output. "Brief me on Acme" → Oracle brief; "what's renewing" → Radar renewal. That's intent→capability, and it lives in each skill's `protocol.md` + synonym map. This file never touches it.

Test: if the rule answers *"which tool do I call for this data?"* it belongs here. If it answers *"what does the user want?"* it belongs in the skill.

---

## Read-routing table

Every data read routes to its authoritative tool. These rules replace per-skill query-building — the charter's "what gets deleted" list is the same set of rules, now living here instead of in capability files.

| Data need | Route to | Never |
|-----------|----------|-------|
| Account attributes (ARR, CSM, Stage, Renewal Date, Segment) | `resolve_customer` **attributes mode** — resolution + C360 attributes in one call | Multiple `run_query` sub-calls per attribute |
| Health score | `ask_scorecard` — overall + **trend** (history) + measure breakdown + root cause | Point-in-time score render with no trend |
| Timeline / activities | `fetch_timeline_activity_list` with `contextual_user_query` (semantic, relevance-ranked, auto-RAG, ~1yr default) | `run_query` on timeline; keyword topic filtering — both against MCP guidance |
| CTAs (incl. linked objects) | `fetch_cta_list` with `linked_object` + `IsClosed=false` | `run_query` (can't fetch CTA linked objects) |
| Customer contacts / stakeholders | `fetch_customer_contacts` (permissions enforced API-side — don't gatekeep, never refuse) | `run_query` on contacts |
| Person resolution (name → GSID) | `resolve_user` (ranked fuzzy + disambiguation) | Raw `gsuser` query first — but see error-handling A4: common first names truncate, fall back to last-name CONTAINS |
| Self-scoped reads ("my CTAs / activities / contacts") | `literal: "CURRENT_USER"` in the filter — server resolves the caller, unspoofable | GSID injection from the identity query for self-scope (keep identity query for tier/org only — see identity.md) |
| Cross-account Staircase signal **filtering** | Gainsight `run_query` on synced `Staircase_*__gc` fields (cheap, no truncation) | N× `staircase_analyze_account` to filter a book — see B7 below |
| Field/schema uncertainty mid-query | `get_object_metadata(object, <query as context>)` as recovery — see error-handling A1 | Bulk-loading metadata up front |

---

## B7 — Staircase ↔ Gainsight synced-field routing (the anchor)

**The mechanic:** Staircase pushes its signals onto the Gainsight **Company** object as queryable fields — e.g. `Staircase_Account_Dark__gc`, `Staircase_No_QBR__gc` (illustrative — **verify the live set per tenant**, don't trust a hardcoded list; `get_object_metadata("company", "staircase")` confirms what's actually synced). So the same signal is reachable two ways, and they cost very differently.

**The rule — route by cardinality:**

- **Many accounts (filter / segment a book):** read the synced `Staircase_*__gc` fields via Gainsight `run_query`. One cheap call, no truncation, no per-account analyst spin-up. This is how you answer "which of my accounts are dark" or "show me no-QBR risk across the segment."
- **One account (the narrative — *why*):** `staircase_analyze_account` / `staircase_query` for the reasoning, evidence, and recency. Single account only. This is the drill-down on the few that the filter surfaced.

**The decision in one line:** *filter wide on Gainsight synced fields, drill deep on Staircase narrative.* Never invert it — fanning `analyze_account` across a book to filter is the expensive mistake B7 exists to prevent (and it truncates, so it silently under-counts).

**Why it's shared, not in a skill:** Oracle's brief reads one account's Staircase narrative (drill). Radar's plate filters a whole book's flags (filter). Same routing rule, opposite ends — exactly the thing that should be decided once, here, not twice in two capability files.

---

## Recency on Staircase analyses (Bluhm B2 — applies wherever Staircase narrative is read)

Staircase Risk and Expansion analyses are produced by **independent analyst agents** that don't reference each other, and Staircase exposes no analysis timestamp. So:

- Ask each analysis for the **date range of its cited evidence** — that's the only recency proxy available.
- When two analyses disagree about the same stakeholder/account, the one anchored on **more recent evidence** wins.
- **Risk-weighted readiness:** discount expansion readiness by 50% when risk ≥ 4. A hot expansion signal sitting on top of unaddressed risk is not as ready as it reads.

(Full B2 application detail lives with Radar's expansion capability; the routing-level rule is here because any skill reading Staircase narrative inherits it.)

---

*2026.06.13 — Created as shared/source-routing.md (broadened from the charter's planned staircase-routing.md per the shared-vs-skill line). Read source-of-truth for the stack: account attributes, health, timeline, CTAs, contacts, person resolution, self-scoped reads, and the B7 Staircase↔Gainsight synced-field cardinality rule. Anchors Bluhm B7 + B2 recency. Capability/intent routing stays per-skill.*
