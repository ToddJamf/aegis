# Radar — Capability: My Plate / Top Accounts
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/capability-plate.md
#
# CSM: "what's on my plate", "top accounts this week", "what should I work on"
# FLL: "what should I work on" → shows team escalations view (routes to heat-map for full picture)
# Data routing (incl. Staircase): shared/source-routing.md. Output/footer: shared/output-discipline.md.

---

## CSM — My Plate

**Trigger:** "what's on my plate" / "top accounts this week" / "what should I work on" — CSM tier.

### Pagination

Use `identity.bookSize`:
- ≤100 accounts: single query, `limit=100`, sort `RenewalDate ASC` / `Arr DESC`.
- 101–200 accounts: two pages in parallel, merge, then rank.
- >200 accounts: two-page fetch. Note: *"Your book is [N] accounts — showing highest-priority 200 sorted by renewal date and ARR."*

### Query

```python
run_query(
    object_name="company",
    select=[
        "Name", "Gsid", "Customer_Category__gc",
        "Renewal_Status__gc", "RenewalDate", "Arr", "ARR_Band__gc",
        "Overall_Score__gc", "CSM_Sentiment__gc",
        "Churn_Risk_Identified__gc", "Open_CSE_Risk_Mitigation__gc",
        "Last_Timeline_Entry_Engagement__gc",
        "Last_Timeline_Entry_CSM_Email__gc",
        "Engaged_This_Period_by_CSM__gc",
        "Engagement_Model__gc",
    ],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [<requester.Gsid>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    sort_by=[{"sortField": "RenewalDate", "sortOrder": "asc"}],
    limit=100
)
```

### B1 — Unified Staircase fan-out (book / plate pulls)

When the plate pull layers Staircase insight flags onto the book, do it in **one** `staircase_query`
returning the priority fields AND the insight-flag booleans per account — not a book pull plus N flag
pulls. Collapse the fan-out into a single call.

Two gotchas, both load-bearing:

- **"My accounts" does NOT auto-scope in Staircase.** Staircase has no notion of the connected CSM's
  book — a bare "my accounts" query returns the whole tenant. Pass the resolved account set
  (names/ids from the Gainsight book pull) explicitly into the Staircase query. Never assume scope.
- **Portfolio-wide flag queries truncate.** A single Staircase query spanning a full book silently
  caps its result set, so any flag **intersection** ("dark AND no-QBR") under-counts without erroring.
  For filtering a whole book by flags, prefer the Gainsight synced `Staircase_*__gc` fields per
  `shared/source-routing.md` B7 (filter wide on Gainsight, drill deep on Staircase). Use the unified
  Staircase fan-out for the priority slice the plate actually renders (top 3–5 per tier), not the book.

---

### Two-tier output

**Tactical (act this week):** Escalated status · renewal ≤60 days · sentiment D/F. Top 3–5.

**Strategic (invest this week):** Renewal 60–180 days · sentiment C or stale · Grow accounts going quiet · high-ARR Retain past cadence. Top 3–5.

### Action-cue rules

One line per account. Template: `[Lead signal] → [Action verb] [timeframe]. [One reason grounded in data.]`

**Tactical cues:**

| Signal | Cue |
|--------|-----|
| Late Escalated, renewal past due | "Late Escalated — renewal is past due → call today. Understand what it takes to close before this converts to churn." |
| Escalated + Sentiment D/F + renewal ≤30d | "Escalated, Sentiment [X], renewal in [N] days → call this week. Don't let this enter formal review without a conversation." |
| Escalated + renewal ≤60d | "Escalated + renewal in [N] days → get on a call and understand the customer's position before the clock runs out." |
| Escalated, no near-term renewal | "Escalated → call to understand current position. Own the narrative before it owns you." |
| Renewal ≤30d, not escalated | "Renewal in [N] days → initiate the renewal conversation now. Still time to get ahead of it." |
| Sentiment D/F + renewal ≤60d | "Sentiment [X] + renewal in [N] days → start a recovery conversation this week, not next." |

**Strategic cues:**

| Signal | Cue |
|--------|-----|
| Sentiment D + ARR ≥$100K | "Sentiment [X] on $[ARR] → understand what's broken. Last touch [N] months ago — needs a real conversation, not a check-in." |
| Last engagement > 2× category cadence | "Last touch [date] — [N] days past [category] cadence → get back on the calendar this week." |
| Grow + last engagement >45 days | "Grow account, [N] days quiet → one email or short call resets the cadence clock." |
| Renewal 60–180d, no other signals | "Renewal [date] ([N] days) → early touch while there's still runway." |

**Register:** direct, verb-first, specific. No corporate hedging.

**React accounts:** Last touch = MAX(`Last_Timeline_Entry_Engagement__gc`, `Last_Timeline_Entry_CSM_Email__gc`). Note: *"React cadence is CTA-driven — email counts as valid touch for UpMarket KPI."*

### B6 — Cleanup-before-create gate (READ-ONLY)

Before the plate piles on new work, surface existing hygiene debt so the user clears the deck first.
This is **read-only in Radar** — Radar shows the debt, it does not close, reassign, or edit anything
(Phase 1 write block holds). Three checks across the book in scope:

- **Past-due >90d** — open CTAs / success-plan actions with `DueDate` more than 90 days past.
- **Stale-update >90d** — accounts whose last timeline entry (`Last_Timeline_Entry_All__gc`) is older
  than 90 days.
- **Orphaned artifacts** — CTAs / success plans / activities owned by a user who has left
  (`gsuser.IsActiveUser=false` on the owner). These rot silently and skew compliance.

Render as a short "Clear the deck first" strip above Tactical when any check fires. One line each:
*"[N] past-due items >90d · [N] accounts un-touched >90d · [N] artifacts owned by departed users →
clean these before adding new work."* Omit the strip entirely if all three are clear. Don't block the
plate — surface, don't gate.

### Output

HTML artifact. Sections in order: Clear-the-deck strip (B6, if any) · Tactical · Strategic.

---

## FLL — Team Plate

**Trigger:** "what should I work on" / "top accounts" — FLL tier.

FLL my-plate is a lightweight redirect. Render the top 3–5 escalations from `identity.teamGsids` as a quick list, then offer: *"For the full team risk picture, try 'show me the team heat map'."*

```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Csm__gr.Name AS CSM",
            "Renewal_Status__gc", "RenewalDate",
            "Churn_Risk_Level__gc", "Open_CSE_Risk_Mitigation__gc",
            "Overall_Score__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": identity.teamGsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Renewal_Status__gc", "operator": "IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",   # Escalated
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],  # Late Escalated
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=5
)
```

Output: chat list. Offer heat-map for full picture.

---

*2026.06.13 — Migrated onto shared layer (header CalVer; data routing → source-routing, output → output-discipline). Added B1 unified Staircase fan-out (one staircase_query for priority fields + insight booleans; "my accounts" doesn't auto-scope + portfolio flag queries truncate → filter wide on Gainsight synced fields per source-routing B7). Added B6 cleanup-before-create gate, READ-ONLY (past-due >90d · stale-update >90d · artifacts owned by departed users → "clear the deck first" strip).*
*2026-06-03 (v2.0) — Ported from Gainsight Sentinel v3.7 references/my-book.md (my-plate section).*
