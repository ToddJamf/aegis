# Radar — Capability: Expansion Pipeline
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/capability-expansion.md
#
# Tab 2 of the Radar artifact. Full book, E1–E4 scoring, two-pass approach.
# Uses company and relationship data already fetched by capability-renewal.md.
# Load scoring formulas from references/scoring.md.
# Load HTML artifact template from references/artifact.md.

---

## Trigger

Expansion tab only: "expansion pipeline", "which accounts should I expand", "where should I expand".
Full Radar (both tabs): bare "show my radar" / "radar" invocation.

---

## Data source

Tab 2 reuses company and relationship data from the renewal query (fired in `capability-renewal.md`). No additional company query at initial load.

**Two-pass approach — initial render:**

**Pass 1 — over-utilized accounts:** Extract from relationship aggregation (`over_utilized=true`). These always float to the top of Tab 2 regardless of expansion score. Cap at 10 shown initially.

**Pass 2 — top readiness accounts:**
```python
run_query(
    object_name="company",
    select=[
        "Name", "Gsid", "Csm__gr.Name AS CsmName",
        "Arr", "ARR_Band__gc",
        "Expansion_Rediness_Level__gc",
        "Overall_Score__gc", "Staircase_Overall_Score__gc",
        "Auto_Renewal__gc",
    ],
    where=[
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "A"},
        {"name": "True_Customer__gc", "operator": "EQ", "value": [True], "alias": "B"},
        {"name": <book_field>, "operator": "EQ", "value": [<gsid>], "alias": "C"},
        {"name": "Arr", "operator": "GTE", "value": [10000], "alias": "D"},
        {"name": "Gsid", "operator": "NOT_IN", "value": [<over_utilized_gsids>], "alias": "E"}
    ],
    where_filter_expression="A AND B AND C AND D AND E",
    sort_by=[{"sortField": "Expansion_Rediness_Level__gc", "sortOrder": "desc"}],
    limit=40
)
```

Pull relationship rows for Pass 2 GSIDs. Reuse Pass 1 relationship data for overlap accounts.

**Total initial load:** up to 10 (over-utilized) + 40 (top readiness) = 50 accounts.

**"Show all" button:** `sendPrompt("show all expansion accounts")` → full book query. Avoids overhead on initial load.

---

## Scoring

Load `references/scoring.md` for full E1–E4 formulas.

**Scoring inputs:**
- E1: expansion readiness (Expansion_Rediness_Level__gc)
- E2: over-utilization (active devices vs paid; enrolled vs licensed)
- E3: account health (Overall_Score__gc)
- E4: Staircase expansion signal (Staircase_Overall_Score__gc — only if Staircase connected)

**E4 null handling:** If Staircase not connected, E4 = null. In JS: `var e4 = (staircaseConnected && e4Score !== null) ? e4Score : 0;` Never `null + number` → NaN.

**Verdict:** expand-strong / expand-mid / expand-low. Over-utilized → always expand-strong.

---

## Render

Tab 2 of the Radar artifact. Load `references/artifact.md` for full HTML template.

**Sort order:** Over-utilized accounts first (pinned, regardless of score) → then by expansion score descending.

**Auto-renewal chip:** `Auto_Renewal__gc=true` → "Auto ↑" chip. In Tab 2, signals expansion candidate — auto-renewing healthy accounts are warm expansion calls. Does NOT affect expansion score.

**E4 row:** Hide entirely if Staircase not connected. Do not render "N/A."

**Expand/collapse rows:** Score breakdown + Staircase section + Claude synthesis.

**Staircase on-demand (Layer 2 — fires on row expand):**
```python
matches = staircase_account_lookup(name=account.Name)
# High-confidence match only
result = staircase_analyze_account(
    account_id=matches[0]["account_id"],
    query="Current risk signals, recent sentiment, and any expansion opportunities for this account?"
)
```
If no high-confidence match: *"No Staircase match found for [Account Name]."* Do not collapse section.

**Bundle gap chips:** Products with `paid > 0 AND active30 == 0`. Render on account row. Activation/expansion conversations — do not penalize E2 score.

---

*Radar capability-expansion.md v2.0 (2026-06-03) — Extracted from Radar SKILL.md v1.3. Full scoring spec in references/scoring.md. Full HTML spec in references/artifact.md.*
