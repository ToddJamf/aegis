# Shared — Error Handling
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/error-handling.md
#
# Shared across: Oracle, Mentor, Radar, Pulse
# The stack's error foundation. EVERY skill falls back here — nothing handles errors locally.
# Output rules (footer, voice, anti-AI) live in shared/output-discipline.md and apply to all copy below.
# When a new error pattern surfaces in production, append to the right layer.
#
# Verified against probes 2026-04-29 / 2026-04-30. Escalation foundation added 2026-06-13.

---

## The three layers

Every tool-call failure resolves to exactly one of these. Try them in order.

1. **Recover** — known technical error the skill can fix itself. Correct the payload, retry once, stay silent. (Layers below: §A.)
2. **Degrade** — partial or empty data the user can still use. Render what came back, mark the gaps, never crash. (§B.)
3. **Escalate** — access/auth/permission failure the user can't fix and the skill can't recover. Render the access-escalation block so the user can self-serve a fix from CS Ops. (§0 — the foundation.)

Recover silently. Degrade honestly. Escalate cleanly. Never swallow an error, never dump a stack trace at a user.

---

## §0 — Access escalation (the foundation)

**When:** the connector isn't configured, auth fails, Gainsight denies the data, or the identity query returns no record — anything where the user is blocked and neither a retry nor a degrade will fix it. The Gainsight MCP enforces permissions server-side, so a bounce here is authoritative: the floor is real, and the fix lives with an admin, not in the skill.

**Do:** render the user message, fill the escalation block from session context, route it. Do not retry. Do not guess around the permission. Do not refuse with a dead-end.

### User message

Three variants — pick the one that matches the failure. Fill `[Account]` and `[skill]` from context.

**Permission denied / no access:**
> "I can't pull data for **[Account]** — looks like it's outside my access scope. Ping **#help-gainsight** and someone can dig in."

**Account or user not found:**
> "I couldn't find **[Account]** in Gainsight. Check the spelling or try a partial name — if it should exist, ping **#help-gainsight**."

**Tool / connector failure:**
> "Something went sideways pulling data for **[Account]**. Try again in a moment — if it keeps happening, ping **#help-gainsight**."

Tone stays warm and internal. Never blame the user. Never over-apologize. Never dump a technical error. `[skill]` is filled by the calling skill — "Oracle", "Radar", "Pulse", etc.

### Escalation block (the admin-facing part — copy-paste ready)

```
Gainsight access request — via [skill]
User:    [Name] ([email])
Tried:   [capability] on [account/target, if any]
Result:  [plain failure reason — from the map below]
Time:    [YYYY-MM-DD HH:MM, user's local]
Need:    [admin action — from the map below]
```

Keep the block plain and monospaced — it's diagnostic, not decorative. The personality lives in the user message above it, not here.

### Failure → reason → admin action

Map the actual MCP failure to a plain `Result` and a plain `Need`. The admin should never have to decode an error code.

| What actually failed | `Result` line | `Need` line |
|---|---|---|
| Identity query returns no `gsuser` row for the email | `No Gainsight user record found for this email` | `Create or verify this user in Gainsight User Management` |
| Authenticated, but the data call is permission-denied | `Gainsight denied access — insufficient permissions for [object]` | `Grant this seat access to [object]` |
| Connector not configured / OAuth not set up | `Gainsight connector not configured for this account` | `Help set up the Gainsight MCP connector (see connector-setup)` |
| Connected, authed, but seat returns zero scoped data | `Authenticated, but the seat has no data scope` | `Confirm this user's book / role assignment in Gainsight` |

If the failure doesn't match a row, use `Result: Gainsight request failed — reason unclear` and `Need: Review this user's Gainsight access`, and include the raw error text on a final `Raw:` line so an admin can dig.

### Routing

- **Primary:** `#help-gainsight` (Slack ID `CKBKTL1PD`) — confirmed active 2026-06-13. Paste-and-go, fastest path.
- **Paper trail:** a CS Ops ticket at `jamf.it/csopsticket` — for anything that needs tracking or recurs.

Surface both; let the user pick speed vs. record. **Never auto-send to Slack** — the user copies and posts. The skill drafts; the human sends.

---

## §A — Recover (self-healing technical patterns)

Known errors the skill fixes itself, retries once, and never narrates.

### A1. Self-healing field recovery (P_5xxx dead-field)

A query fails with `P_5005 / P_5026`-class `Column ... does not exist`. The bundled field-registry is the fast path, but it rots — both prior P_5xxx hits were caught by humans. Make recovery automatic:

1. Catch the dead-field error.
2. Call `get_object_metadata(<object>, <the user's query as context>)` — it returns a relevance-ranked field subset for the object.
3. Correct the payload from `fields.rows` (trust it over `queryGuidance.defaultSelect` — the latter is partly aspirational).
4. Retry once.
5. Flag the registry entry stale (note it in the response's internal trail, not to the user) so the bundled cache gets fixed on the next maintenance pass.

Registry stays the efficiency cache; metadata is the recovery path. Don't bulk-load metadata up front — it's the fallback, not the default.

### A2. Truncated responses

| Source | Trigger | Mitigation |
|--------|---------|------------|
| `get_object_metadata("company")` | 70k chars | Parse from saved file path (response includes path); cache parsed schema for session |
| `fetch_cta_list` (no filter) | 174k for large accounts | Always include `IsClosed=false`; constrain `select` |
| `fetch_timeline_activity_list` (large account) | 100k+ chars | Always pass `limit=10`; if still truncated, response is saved to a Mac host path — use `Read` tool (not bash) |
| `resolve_user("Tim")` (common first name) | 75k for common first name | Skip `resolve_user`; `run_query` on `gsuser` with last-name CONTAINS filter |

When a saved-file path is returned, use the **`Read` file tool** on the saved path — it is a Mac host path, not accessible from the bash sandbox. Do not use bash or Python to open it.
```
Read(saved_path)  # returns file contents directly; parse JSON from the result
```

### A3. Tenant schema drift — phantom fields (Timeline)

`fetch_timeline_activity_list` errors with `P_5005: Column t1.<X> does not exist`. Jamf corrections:
- Filter on `GsCompanyId` (NOT `CompanyId`)
- Author display: `GsCreatedByUserId__gr.Name AS AuthorName` (NOT `AuthorName`)
- Notes: `NotesPlainText AS Notes` (Notes is RICHTEXTAREA, can't filter)
- Skip `Sentiment`; use `Ant__Sentiment__c` if needed

Always pass explicit select — see `field-registry.md`. Trust `fields.rows` from metadata over `queryGuidance.defaultSelect`.

### A4. Resolution & lookup errors

#### Account disambiguation — multiple matches

When `resolve_customer` returns >1 result, resolve in this order:

**Step 1 — Account team check (primary path).**
Fetch account team fields for each match:
```python
select = ["Gsid", "Name",
          "Csm", "Account_Owner__gc",
          "Sales_Engineer__gc", "Renewal_Rep__gc",
          "Executive_Sponsor__gc"]
```
Check each field against `requester_gsid` (from identity.md).

- **Exactly one match** has the requester on the team → proceed with that account.
  If the match has an IA sibling, note it: *"Pulled [Account] — there's also a sibling IA account. Combined brief?"*
- **Multiple matches** have the requester on the team → narrow to those, go to Step 2.
- **No match** has the requester on the team → go to Step 2.

**Step 2 — Context narrowing (non-account-team path).**
Use available context to eliminate false matches: region, segment, ARR band, CSM name if mentioned.
Reduce the list as far as possible. If one remains, proceed.

**Step 3 — Ask plainly.**
If still ambiguous, show the short list and ask directly. Keep it clean:

```
Found [N] accounts matching "[name]" — which one?

1. [Account Name] · [ARR band] · [Region] · CSM: [Name]
2. [Account Name] · [ARR band] · [Region] · CSM: [Name]
   ↳ + [Account Name (IA)] — these two can be pulled as a combined brief
```

Never guess. Never proceed silently on a multi-match. Cache the choice for the session.

---

**Common first-name truncation:** `resolve_user` does fuzzy substring; "Tim" matches too many. Default to `run_query` on `gsuser` with last-name CONTAINS + `IsActiveUser=true`.

**`P_5068 Invalid Lookup name`:** `__gr.Name` used on a PICKLIST/MULTISELECTDROPDOWNLIST. Only LOOKUP fields support it.

| dataType | Pattern |
|----------|---------|
| LOOKUP | `Field__gr.Name AS Alias` |
| PICKLIST / MULTISELECTDROPDOWNLIST | `Field` direct (auto-returns `Field_PicklistLabel`) |

### A5. `where_filter_expression` overrides implicit AND

**Verified bug 2026-04-30.** When `where_filter_expression` is included, it REPLACES the default AND-join. Forgetting to anchor a critical filter (Csm scope, Account_Status) silently returns broader results.

```python
# WRONG — Csm and Account_Status anchors get dropped
where_filter_expression = "(A OR B)"
# RIGHT — every filter aliased and in the expression
where_filter_expression = "X AND Y AND (A OR B)"
```

If only AND-logic is needed, omit `where_filter_expression` entirely.

### A6. Verified bulk-call safe limits (Jamf, 2026-04-30)

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

## §B — Degrade (partial / empty data is not an error)

### B1. Empty results — not failures

| Source | Empty signal | Render as |
|--------|--------------|-----------|
| `staircase_query` | `{"answer": "No reply found", "evidences": []}` | "No recent Staircase signal" |
| `ask_scorecard.measures_flat` | `[]` | Render overall score; note "measure-level breakdown unavailable" |
| `ask_scorecard.overall_score_history` | `[]` | Skip trend line |

Empty Staircase = no signal (neutral), NOT no risk (positive). Different.

### B2. Mostly-null company records

Accounts with Customer_Category populated, everything else null. Render `—` or "Not set" per field — never crash the row. Empty-string `""` on date fields = null. Don't exclude null-renewal accounts from at-risk scoring. In FLL roll-ups, surface as "data quality issue — ask CSM to populate."

### B3. Inactive / OOO users

`gsuser.IsActiveUser=false` or `Out_of_Office__gc=true` → exclude from "needs work" lists and team-load views by default; include only if asked. If the *asker* is OOO: *"You're marked OOO — still pulling your book?"*

### B4. Customer is Company vs Relationship

`resolve_customer` returns both arrays. Default to Company unless the user says "relationship." Cache the choice. Surface it: *"Resolved to company Acme, Inc. — say 'relationship' if you meant a sub-product."*

### B5. Scorecard sync lag

`ask_scorecard` can return a score stale relative to a timeline milestone already recorded (e.g., an account returns `overall_score: 50 / C` while its timeline shows "Health score changed from C to B" the same day). If the timeline has a score-change milestone within 24h, note the transition; don't suppress:
```
Health: C (upgrading to B per today's milestone — scorecard sync may lag a few hours)
```

### B6. Compliance roll-up incomplete — non-standard Account_Status

The compliance roll-up filters `Account_Status__gc = Current Customer`, silently excluding null/non-standard status (reseller, distributor, test, $0 ARR). If it returns data for <20% of CSMs in scope, surface a hygiene flag rather than a misleading %:
```
⚠️ Compliance data incomplete — only {N} of {total} CSMs have accounts matching standard
Account_Status. Book may include reseller, distributor, or test accounts outside the
compliance filter. Verify with CS Ops before relying on these numbers.
```
Never compute a % from partial data.

---

## Module shape (conceptual)

`safe_call()` is shorthand for the discipline in this file — the pre-call checks and post-call handling wrapped around any MCP tool call. Not a real callable. Before a call: scope anchors, explicit select, safe batch size. After: recover (§A) → degrade (§B) → escalate (§0), in that order.

When a new pattern surfaces in production: append it to the right layer. Capability modules don't change — fixes live here.

---

*2026.06.14 — Bug fix A2: `fetch_timeline_activity_list` added to truncation table with `limit=10` enforcement. Saved-file parse instruction corrected from bash/Python to `Read` tool (Mac host path not accessible from bash sandbox — confirmed in live test).*
*2026.06.13c — A4 account disambiguation rewritten: account team membership (Csm, AE, SE, Renewal Rep, Exec Sponsor) is the primary resolution path. Non-team users fall to context narrowing, then plain ask with differentiating list.*
*2026.06.13b — §0 user message locked: three scenario variants (permission denied, not found, tool failure). Short, warm, always actionable.*
*2026.06.13 — Restructured into three layers (Recover / Degrade / Escalate). Added §0 access-escalation foundation — the stack-wide fallback all skills drop to, with copy-paste CS Ops block routed to #help-gainsight (CKBKTL1PD, confirmed) + jamf.it/csopsticket. Added A1 self-healing field recovery (get_object_metadata on P_5xxx). Header updated to CalVer + full skill list; output rules delegated to output-discipline.md. Prior technical sections preserved, reorganized under §A/§B.*
*2026.06 (v2.0) — Original 12-section technical error catalog. Verified against probes 2026-04-29/30.*
