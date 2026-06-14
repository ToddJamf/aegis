# Oracle — Glossary / Terminology lookup
# Aegis stack 2026.06.13
# Bundled (local). Load when a user asks for a definition or explanation of a Gainsight or Staircase concept.

**Permission:** ALL users — CSM, Leader, AND Not-CS. Does not expose customer data.

**Triggers:** "what is a [term]" / "what does [term] mean" / "explain [concept]" / "what's the difference between [X] and [Y]" / "tell me about [field/value]".

---

## Terminology file routing

Load the correct file based on the requesting user's tier (resolved in Step 1):

| Tier | Load |
|------|------|
| Not-CS | `references/terminology-public.md` only — concept definitions, no thresholds or playbook detail |
| CSM / Leader | `references/terminology-cs.md` — full detail including cadences, field values, escalation tiers |

If Step 1 hasn't run yet (e.g., user triggered Glossary before any data op), default to `terminology-public.md` and note that CS users can get more detail by asking again after identity is confirmed.

---

## Sequence (~0 tool calls, doc lookup only)

1. Load the appropriate terminology file (per routing above).
2. Match the user's term against the glossary content — synonym map, field values, Gainsight terms, Staircase terms.
3. Return a focused 2-4 sentence definition. For tier/segmentation terms (Customer Category, Renewal Status, ARR Band), include the value table. For multi-step concepts (CTA, Success Plan, Scorecard), include the definition plus how it shows up in Oracle.

---

## Render

Plain chat. Answer first, context second.

```
**[Term]** — [1-2 sentence definition].

[Optional: table of values, examples, or how it appears in Oracle]
```

### Example: "what is Customer Category?"

```
**Customer Category** — Jamf's CS tiering field (`Customer_Category__gc`), used
as the KPI dimension for engagement cadence.

Values and cadences:
  • Spotlight  — Top-priority strategic, bi-weekly
  • Retain     — Established, maintain, every 3 weeks
  • Grow       — Expansion target, monthly
  • React      — Lower-touch / reactive, ~6mo CTA-driven

In Oracle, this drives the Customer Category line in a pre-call brief.
```

### Example: "what is a CTA?"

```
**CTA (Call to Action)** — Gainsight's name for an action item / task / issue
assigned to a CSM. Lives in the Cockpit. Has a Type, Priority, Status, and Owner.
May contain sub-tasks (`cs_task` records).

In Oracle, CTAs surface in pre-call briefs (top 3 open).
```

---

## Edge cases

- **Term not in glossary** → *"I don't have a definition for that in the bundled glossary. Try the Confluence canonical page, or ask your CS Ops partner."*

- **Compound query** (e.g., "what is a CTA and how do I close one") → Answer the term definition. For the process part: *"For how-to questions about the Gainsight UI, that's outside Oracle — check Gainsight's documentation or ask your CS Ops team."*

- **Ambiguous term** (e.g., "what does score mean" — health score? sentiment grade?) → Ask which one, list the candidates.

- **Data query disguised as glossary** (e.g., "what is Customer Category for Acme") → Answer the term definition only. The specific account value is a data query — for a factual account lookup, ask the Gainsight connector directly (or Oracle for a full brief).

- **CS-gated term asked by Not-CS user** (e.g., "what are your escalation thresholds") → Provide the concept definition from `terminology-public.md`. Do not reveal internal cadence thresholds or playbook actions. *"I can explain the concept — for Jamf's internal thresholds, reach out to your CS Ops partner."*

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
