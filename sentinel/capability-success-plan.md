# Sentinel — Capability 10: Draft Success Plan
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-success-plan.md

Load on demand when user asks to create or draft a success plan.

**Triggers:** "create a success plan for [customer]", "draft a plan for [customer]", "success plan for [customer]", "I need a plan for [customer]".

**Permission:** CSM, CSE, Leader. Not-CS → refused: *"Success plan drafting requires a CS team account. You can still look up account details (try 'when does Acme renew?')."*

---

## Signal pull sequence

Fire in parallel before drafting:

1. `resolve_customer` — get GSID
2. `run_query` on `company` — ARR, renewal date/status, category, sentiment, CSM
3. `ask_scorecard(company_ids=[gsid])` — health score
4. `fetch_cta_list` where `IsClosed=false AND CompanyId=<gsid>` — open CTAs become action items
5. `fetch_success_plan_list` where `CompanyId=<gsid> AND IsClosed=false` — check for existing plans
6. `fetch_timeline_activity_list` where `GsCompanyId=<gsid>`, last 90 days, limit 10
7. `staircase_query("Current risk, priorities, and relationship status for {customer_name}?")` — in parallel

If active plan exists: *"There's already an active success plan ([Name], [%] complete, due [date]). Do you want to update it or draft a new one?"* Wait for confirmation.

---

## Plan type inference

Infer from signals. Do not ask unless ambiguous.

| Signal | Inferred type |
|--------|--------------|
| Health score D/F, or Staircase churn risk | Risk / Save |
| Renewal ≤90 days | Renewal |
| New account or CSM transition in last 90 days | Onboarding |
| Open CTAs about adoption gaps or feature rollout | Adoption |
| No clear signal | Ongoing Success |

If ambiguous: pick more urgent, note it. *"Drafting as a Renewal plan given the 60-day renewal window — let me know if you want a different type."*

---

## Synthesis rules

- Lead with the customer's most pressing signal (Staircase risk > renewal urgency > adoption gap > general health)
- Ground every CTA and task in data — cite the signal ("Deployed% is 34% — adding an adoption milestone")
- Max 3–5 CTAs. Don't pad.
- Each CTA: 1–2 tasks max. Verb-driven, completable in ≤2 weeks.
- Avoid generic items ("Follow up," "Check in") — if not tied to a specific signal, drop it.
- Thin data (new account, no timeline, no Staircase): draft a 2-CTA skeleton, flag what's missing.

---

## Output format (paste-ready for Gainsight entry)

```
SUCCESS PLAN DRAFT — [Customer Name]
Type: [Plan type]
Goal: [1 sentence — what does success look like at plan close?]
Target close: [date — typically renewal date or 90 days out]

── CTA 1: [Name] ──
Priority: [High / Medium]
Due: [date]
Playbook: [if applicable]
Why: [1 sentence tied to data]

  Task 1: [verb-driven, specific]
  Due: [date]

  Task 2: [verb-driven, specific — if needed]
  Due: [date]

── CTA 2: [Name] ──
[same structure]

...

NOTES FOR CSM:
[1–3 bullets on context that isn't in the plan — Staircase risk framing, exec sponsor status, pending questions]
```

ARR: do not include unless requester owns the account or is in the manager chain (Cap 10 = bulk/list capability — apply ARR restriction rules from protocol.md).

Output in chat, not artifact. Direct Gainsight writes deferred to stage 2.

---

## CSE variant

When requester is CSE tier:
- Include a "CSE Resolution" CTA if `CSE_Request__gc` is populated — customer's own words become the CTA context
- Ask Jamf doc lookup (same as Cap 3 Batch 3): scan CTA topics, formulate contextual query, surface 3–5 relevant public docs as appendix
- Omit renewal context block (CSE plans are technical, not renewal-driven)

---

## Graceful degradation

- **Staircase not connected:** draft from Gainsight only. Note: *"(Staircase not connected — risk signals not included)"* at top.
- **No open CTAs:** draft 2 placeholder CTAs based on health score + renewal proximity. Flag: *"(no open CTAs — suggested based on account signals, not active work items)"*
- **Customer not found:** follow `error-handling.md` Section 3.
- **Active plan exists:** surface it first, ask before drafting new.
