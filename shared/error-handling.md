# Shared — Error Handling
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/error-handling.md
#
# Shared across: Sentinel, Pulse, Radar
# Reactive layer. When a new error pattern surfaces in production, append a section here.
# Capability modules don't change — fixes live here.
#
# Verified against probes 2026-04-29.

---

## 1. Truncated responses

| Source | Trigger | Mitigation |
|--------|---------|------------|
| `get_object_metadata("company")` | 70k chars | Parse from saved file path (response includes path); cache parsed schema for session |
| `fetch_cta_list` (no filter) | 174k for large accounts | Always include `IsClosed=false`; constrain `select` |
| `resolve_user("Tim")` (common first name) | 75k for common first name | Skip `resolve_user`; use `run_query` on `gsuser` with last-name CONTAINS filter |

When a saved-file path is returned, parse with:
```python
raw = json.load(open(saved_path))
data = json.loads(raw[0]['text']) if isinstance(raw, list) else raw
```

---

## 2. Tenant schema drift — phantom fields (Timeline)

`fetch_timeline_activity_list` errors with `P_5005: Column t1.<X> does not exist`.

**Required corrections for Jamf:**
- Filter on `GsCompanyId` (NOT `CompanyId`)
- Author display: `GsCreatedByUserId__gr.Name AS AuthorName` (NOT `AuthorName`)
- Notes: `NotesPlainText AS Notes` (Notes is RICHTEXTAREA, can't filter)
- Skip `Sentiment` field — use `Ant__Sentiment__c` if needed

**Always pass explicit select** — see `field-registry.md`. Trust `fields.rows` from metadata over `queryGuidance.defaultSelect` (latter is partly aspirational).

---

## 3. Resolution & lookup errors

**Ambiguous match (>1 result):** Hit on `resolve_user("Chris")` (7+ matches), `resolve_customer("Nike")` (Nike Inc + Nike Japan + false matches).

Handler: surface matches with disambiguators (Email, Role, Customer Category, ARR), prompt to pick. Cache choice for session. Filter to exact-prefix matches first.

**Common first-name truncation:** `resolve_user` does fuzzy substring. "Tim" matches too many. Default to `run_query` on `gsuser` with last-name CONTAINS + `IsActiveUser=true`.

**`P_5068 Invalid Lookup name`:** Using `__gr.Name` on a PICKLIST or MULTISELECTDROPDOWNLIST field. Only LOOKUP fields support that pattern.

| dataType | Pattern |
|----------|---------|
| LOOKUP | `Field__gr.Name AS Alias` |
| PICKLIST / MULTISELECTDROPDOWNLIST | `Field` direct (auto-returns `Field_PicklistLabel`) |

---

## 4. Inactive / OOO users

`gsuser.IsActiveUser=false` or `Out_of_Office__gc=true` → exclude from "needs work" lists and team load views by default. Include only if user explicitly asks.

If the *asker* is OOO, surface in response: *"You're marked OOO — still pulling your book?"*

---

## 5. Customer is Company vs Relationship

`resolve_customer` returns both arrays. Default to Company unless user says "relationship." Cache choice. Surface what was used: *"Resolved to company NIKE, Inc. — say 'relationship' if you meant a sub-product."*

---

## 6. Empty results — not errors

| Source | Empty signal | Render as |
|--------|--------------|-----------|
| `staircase_query` | `{"answer": "No reply found", "evidences": []}` | "No recent Staircase signal" |
| `ask_scorecard.measures_flat` | `[]` | Render overall score; note "measure-level breakdown unavailable" |
| `ask_scorecard.overall_score_history` | `[]` | Skip trend line |

Empty Staircase = no signal (neutral), NOT no risk (positive). Different.

---

## 7. Mostly-null company records

Hit on accounts with Customer_Category populated, everything else null.

Handler: render `—` or "Not set" per field, never crash the row. Empty-string `""` on date fields = null. Don't exclude null-renewal accounts from at-risk scoring. Surface in FLL roll-up as "data quality issue — ask CSM to populate."

---

## 8. Verified bulk-call safe limits (Jamf, 2026-04-30)

| Tool | Safe size per call |
|------|-------------------|
| `ask_scorecard` | 10 company_ids |
| `run_query` aggregation with `IN [N]` | 10 GSIDs |
| `fetch_cta_list` with `IsClosed=false` | Single customer |
| `fetch_success_plan_list` | Single customer |
| `fetch_timeline_activity_list` | Single customer, limit ≤10, explicit select |
| `fetch_timeline_activity_list` team-scoped | 7 CSMs, last 60d, limit ≤40 |
| `run_query` compliance roll-up | 7 CSMs in IN clause |
| `run_query` escalation query | 7 CSMs, limit ≤25 |

Larger batches: chunk into groups of 10 and merge.

---

## 9. `where_filter_expression` overrides implicit AND

**Verified bug 2026-04-30.** When `where_filter_expression` is included, it REPLACES the default AND-join. Forgetting to anchor a critical filter (e.g., Csm scope, Account_Status) silently returns broader results.

```python
# WRONG — Csm and Account_Status anchors get dropped
where_filter_expression = "(A OR B)"  # ← drops any unaliased filters

# RIGHT — every filter has alias and is in expression
where_filter_expression = "X AND Y AND (A OR B)"
```

If only AND-logic is needed, omit `where_filter_expression` entirely. Use it only when OR-logic is required.

---

## 10. Scope gate: CS-team-only capabilities

If identity = Not-CS: do not run CS-only capabilities. Render the polite refusal from protocol.md. Cache the determination — do not re-check on every query.

Not an error case — a scope rule.

---

## 11. Scorecard sync lag

`ask_scorecard` can return a health score stale relative to timeline milestones already recorded. Observed: Cargill returned `overall_score: 50 / label: C` while timeline showed "Health score changed from C to B" that same day.

**Pattern:** If the timeline contains a score-change milestone within the last 24h, note the transition:
```
Health: C (upgrading to B per today's milestone — scorecard sync may lag a few hours)
```

Render scorecard score + transition note. Don't suppress.

---

## 12. Compliance roll-up incomplete — non-standard Account_Status

The compliance roll-up filters by `Account_Status__gc = Current Customer`. This silently excludes accounts where status is null or non-standard (reseller/distributor, test accounts, $0 ARR).

**Detection:** if compliance roll-up returns data for < 20% of the CSMs in scope, surface a data hygiene flag rather than computing a misleading percentage:

```
⚠️ Compliance data incomplete — only {N} of {total} CSMs have accounts matching standard
Account_Status. Book may include reseller, distributor, or test accounts outside the
compliance filter. Verify with CS Ops before relying on these numbers.
```

Do not silently compute a % from partial data.

---

## Module shape (conceptual)

`safe_call()` is a conceptual pattern representing the set of pre-call checks and response handling steps applied before and after any MCP tool call. It is not a real callable function — the label is shorthand for the discipline described in this file.

When a new error pattern surfaces in production: append a section here. Capability modules don't change.
