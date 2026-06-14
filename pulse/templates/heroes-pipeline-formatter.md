# Pulse — Heroes Pipeline Formatter
# Aegis stack 2026.06.13
# Bundled (local).

---
name: heroes-pipeline-formatter
version: "1.0"
purpose: Chat output format for the heroes_pipeline capability
---

# Heroes Pipeline — Chat Output Formatter

## Output Format

The Heroes Pipeline delivers a **chat table** — not an HTML artifact. This keeps it fast and shareable for community team outreach workflows.

---

## Full Output Template

```
These members are not current Heroes and score ≥ 35 on the Heroes Readiness Score.
Scores reflect community engagement only. Program eligibility and prior application history are not factored in.

| Username | Score | Replies (365d) | Replies (90d) | Solutions | Role |
|---|---|---|---|---|---|
| [username] | [score] | [replies_365d] | [replies_90d] | [solution_milestone] | [role_label] |
... (up to ~16 rows)

Share with the community team to initiate application outreach.
```

---

## Field Guidance

**Score** — Round to one decimal. e.g. `75.0`, `42.3`.

**Role** — Use simplified labels only:
- Verified Contributor
- Partner
- Member

Do not output raw `CAT_ROLES` strings (they will contain long, noisy values).

**Sort** — Highest score at top. Descending.

---

## Edge Cases

**Zero results:**
```
No candidates meet the current threshold of 35. Contact the community team to review the threshold.
```
Do not suggest lowering the threshold yourself. That is a program decision.

**Results exceed 20:**
This should not occur at current threshold (≥ 35 → ~16 candidates). If it does, suggest threshold review rather than outputting a very long table:
```
[N] candidates meet the threshold — more than expected. Consider reviewing the threshold with the community team before sharing. Showing top 20:
[table]
```

**Single result:**
Output normally — one-row tables are fine.

---

## What NOT to do

- Do not output individual scores as an HTML artifact — chat table is the right format for this list size
- Do not include raw `CAT_ROLES` in the table
- Do not suggest lowering the threshold in the empty-state message
- Do not add commentary about individual candidates in the output (beyond the standard table)
- Do not initiate contact or draft outreach messages unless explicitly asked

---

*2026.06.13 — Ported; scrubbed, CalVer.*
