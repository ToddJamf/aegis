# Ask Gainsight — Protocol
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/ask-gainsight/protocol.md
#
# Ask Gainsight is a standalone quick-lookup skill. No tier gate — any user,
# any account, any factual question answerable from Gainsight data.
# Fast, direct, no meeting prep framing.

---

## Does NOT do

- **No account briefs** — narrative pre-call prep → Oracle
- **No 1:1 prep** → Oracle
- **No success plans** → Oracle
- **No operational views** — plate, gaps, compliance, heat-map, portfolio → Radar
- **No ARR analytics** — Snowflake trends → ask-snowflake-analyst
- **No writes** — read-only, always
- **No external sends**

---

## Output discipline

- Answers in 1–3 lines for single-account factual questions
- Multi-account answers: parallel resolution, up to 5 per ask (hard cap)
- Offer brief at end of single-account answers: *"Want a full brief on [Account]? Try 'brief me on [Account]' in Oracle."*
- No preamble, no transition phrase — first line is the answer

**No footer required** — Ask Gainsight is a quick-look tool. Skip the AI disclaimer footer on single-fact answers. Include it on any substantive multi-account output.

---

## Step 0 — MCP inventory

- **Gainsight** (`run_query` callable) — required. If absent: *"Gainsight isn't connected. Go to Cowork → Settings → Connectors → Gainsight → Always allow."* Stop.
- **Staircase** (`staircase_account_lookup` callable) — optional. If absent: skip Staircase; proceed with Gainsight-only answer.

---

## Step 1 — Resolve the account

Use `resolve_customer` for account name resolution. If multiple matches → disambiguation prompt: *"Found [N] matches for '[name]' — which one? [list names + CSM]"*

**IA accounts:** If query returns both an IA and non-IA variant, answer from the account that best fits the question. For financial questions (ARR, renewal date) → IA. For operational questions (health, CSM, products) → non-IA. If ambiguous → surface both answers.

---

## Lookup — single account

Default field set:
```python
run_query(
    object_name="company",
    select=[
        "Name", "Gsid",
        "Csm__gr.Name AS CsmName",
        "RenewalDate", "Renewal_Status__gc",
        "Arr", "ARR_Band__gc",
        "Overall_Score__gc", "CSM_Sentiment__gc",
        "Customer_Category__gc",
        "Last_Timeline_Entry_Engagement__gc",
        "Engagement_Model__gc",
        "Auto_Renewal__gc",
    ],
    where=[
        {"name": "Name", "operator": "CONTAINS", "value": [<name>], "alias": "A"}
    ],
    where_filter_expression="A",
    limit=5
)
```

Answer the question from the returned fields. If the question asks about a field not in the default set, add it to the select before firing.

**ARR:** Unrestricted for all users on single-account lookups. Render full dollar amount.

**Risk / sentiment questions:** If user asks about risk, concerns, or relationship health → use Gainsight fields AND fire Staircase Pattern A in parallel:
```python
matches = staircase_account_lookup(name=customer_name)
# High-confidence match only
result = staircase_analyze_account(
    account_id=matches[0]["account_id"],
    query="Current risk signals and recent relationship health for this account?"
)
```
Answer from Gainsight first, layer in Staircase narrative if available.

---

## Lookup — multi-account (up to 5)

Resolve and query in parallel. Hard cap: 5 accounts per ask.

If user asks for > 5 → *"Ask Gainsight looks up up to 5 accounts at a time. I'll pull these 5 — want me to run the rest separately?"*

**Output:** inline table or list. No artifact for ≤5 accounts.

---

## Glossary — terminology lookup

Trigger: "what is a [term]" / "explain [concept]" / "difference between [X] and [Y]".

Permission: all users. Load `references/terminology-public.md` for general users. Load `references/terminology-cs.md` for CS org users (detected from Gainsight identity if available; default to public if identity not resolved).

---

## Identity resolution (optional)

Ask Gainsight does not require identity resolution for factual lookups. Identity is resolved opportunistically — if the session has already resolved identity from another skill (Oracle, Radar), reuse the cached tier. If not, proceed without identity and treat the user as general access.

Identity resolution only fires if:
- User asks a question scoped to their own book ("who are my renewal accounts this quarter")
- User asks about team data ("show me my team's escalations")

In those cases: load `https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/identity.md` and resolve tier before answering.

---

## Edge cases

- Account not found → *"Couldn't find '[name]' in Gainsight. Check the spelling or try a partial name."*
- Multiple strong matches → disambiguation prompt (see Step 1)
- Field returns null → render "—" not blank. For ARR null: "ARR not on file — may not be synced yet."
- Staircase no match → answer from Gainsight only, no mention of Staircase

---

*Ask Gainsight v1.0 (2026-06-03) — Standalone quick-lookup skill carved from Gainsight Sentinel v3.7 Cap 9 (Ask Gainsight) and Cap 8 (Glossary). No tier gate. All users. Built on Aegis hosted logic framework.*
