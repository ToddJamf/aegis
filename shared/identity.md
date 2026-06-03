# Shared — Identity Resolution
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/identity.md
#
# Shared across: Sentinel, Pulse, Radar
# Update here → all skills pick it up on next activation.

---

## Identity query

```python
run_query(
    object_name="gsuser",
    select=[
        "Name", "Gsid", "Email",
        "Account_Assignment_Resource_Type__gc",
        "CS_Territory_Team__gc", "CS_Territory_Region__gc",
        "Manager__gr.Name", "Manager",
        "IsActiveUser", "Out_of_Office__gc"
    ],
    where=[
        {"name": "Email", "operator": "EQ", "value": [<email>], "alias": "A"},
        {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    limit=1
)
```

Use the session email. Cache result as `identity` for the session — never re-fetch.

---

## Tier detection sequence

Run in order. Stop at first match.

### Step A — Leader check (recursive walk)

**Step A1 — direct reports:**

```python
run_query(
    object_name="gsuser",
    select=["Gsid", "Name", "Account_Assignment_Resource_Type__gc"],
    where=[
        {"name": "Manager", "operator": "EQ", "value": [<user.Gsid>], "alias": "A"},
        {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}
    ],
    where_filter_expression="A AND B"
)
```

**Step A2 — CS role check:**

Quick `run_query` on `company` filtered to `Csm IN [report_gsids]` with `LIMIT 1`. If any result → at least one direct report is CS-tier.

**Step A3 — recursive walk (if needed):**

If direct reports are all non-CSM, walk their reports recursively. Depth cap: 5. Accumulate all descendant Gsids who carry a CS role into `leader.teamGsids`.

- Any CS-role node found → **tier = Leader**. Cache `leader.teamGsids`. Stop.
- No CS-role nodes → continue to Step B.

---

### Step B — CSE check

```
if identity.Account_Assignment_Resource_Type__gc == "CSE":
    tier = CSE
    stop
```

**Field note:** `Account_Assignment_Resource_Type__gc` is a picklist on `gsuser` with values `"CSS"` and `"CSE"`. CSEs carry `"CSE"` (GSID: `1I00ID65UTDE7E0PDICKCFPWZLYI382QGRLA`). Confirmed against live Gainsight (v3.2).

**Do NOT use `Employee_Type__gc`** — values are Individual Contributor / Leadership / Executive only. No CSE value exists on that field.

If null or not "CSE" → fall through to Step C.

---

### Step C — CSM check

```python
run_query(
    object_name="company",
    select=["Gsid"],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [<user.Gsid>], "alias": "A"}
    ],
    where_filter_expression="A",
    limit=1
)
```

- Result → **tier = CSM**. Stop.
- No result → **tier = Not CS**. Stop.

**Do NOT filter by `Status = Active` here.** A CSM whose entire book has non-Active status is still a CSM. Status filtering belongs inside capabilities, not tier detection.

---

### Step D — Not found

Identity query returned no record → **tier = Not found**.

Output: *"Couldn't find you in Gainsight. CS data capabilities require a CS team account, but you can still look up accounts (try 'when does Acme renew?') or ask terminology questions (try 'what is a CTA')."*

---

## Book size (run in parallel with identity query)

```python
run_query(
    object_name="company",
    select=["COUNT"],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [<user.Gsid>], "alias": "A"},
        {"name": "Status", "operator": "EQ", "value": ["Active"], "alias": "B"}
    ],
    where_filter_expression="A AND B"
)
```

Cache as `identity.bookSize`. Used by skill menus and Cap 4 pagination.

For CSE tier: use Csm-owned count for menu display.

---

## Identity confirmation

**Auto-confirm (no prompt):** session email resolves to exactly one user with no collision → proceed silently. Cache `verified_at = now()`. Do NOT prompt "Running as [Name] — is that right?"

**Prompt when:** (a) multiple users match, (b) fuzzy match was used, (c) email was manually overridden. Prompt: *"Running as [Name] ([tier]) — is that right?"*

If the user corrects identity: re-run with corrected email, re-cache.
