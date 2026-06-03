---
name: ask-gainsight
description: "Use this skill for quick, factual lookups from Gainsight — no tier gate, any user, any account. Trigger when someone asks a specific factual question about a named account: 'when does Acme renew?', 'who is the CSM on Acme?', 'what's the health score for Acme?', 'what's the ARR for Acme?', 'is Acme a Spotlight account?', 'what products does Acme have?', 'what is a CTA?', 'what is a success plan?', or any 'what is a [Gainsight term]' question. Also use for looking up 2–5 accounts at once. This skill answers in 1–3 lines — it does not produce a full brief. For pre-call briefs, 1:1 prep, or success plans use Oracle. For renewal triage, plate management, or team views use Radar. For ARR analytics use ask-snowflake-analyst."
---

# Ask Gainsight

*Quick factual lookups from Gainsight. Any user, any account, no setup required.*

---

## On activation

Fetch and follow:

```
https://raw.githubusercontent.com/ToddJamf/aegis/main/ask-gainsight/protocol.md
```

That file governs all lookup logic, multi-account handling, Staircase routing, and glossary behavior. Follow it exactly.

**If the fetch fails:** Answer directly from Gainsight using the default field set (Name, CsmName, RenewalDate, Arr, Overall_Score__gc, CSM_Sentiment__gc, Customer_Category__gc). Surface the fetch failure as a warning: *"⚠ Ask Gainsight couldn't load its full instructions — answering with base fields only."*

---

## Bundled reference files

- `references/field-registry.md` — Gainsight field definitions and safe-select patterns
- `references/terminology-cs.md` — CS terminology (for CS org users)
- `references/terminology-public.md` — CS/Gainsight terminology (for all users)
- `references/arr-policy.md` — ARR render policy (unrestricted for single-account lookups)

---

*Ask Gainsight v1.0 (2026-06-03) — Standalone quick-lookup skill. Carved from Gainsight Sentinel v3.7. Built on Aegis hosted logic framework.*
