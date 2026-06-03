# Pulse — Capability: Unanswered Posts
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-unanswered.md
#
# Triggers: "unanswered questions", "unanswered posts", "what needs a reply",
#           "posts with no answer", "questions without answers"

---

## Definition

An unanswered post is a community question (`CAT_CONTENT_TYPE = 'question'`) with no accepted answer event in `CONTENT_QUESTION_ANSWERED`. Does not include non-question topics.

**Note:** `CONTENT_QUESTION_ANSWERED` validated as firing correctly as of 2026-05-11.

---

## Query logic

Load `references/unanswered.md` for the full confirmed SQL, aging bands, and output spec.

Key elements:
- Source: `CONTENT_TOPIC_CREATED` LEFT JOIN `CONTENT_QUESTION_ANSWERED` on `ID_TOPIC_PUBLIC`
- Filter: `CAT_CONTENT_TYPE = 'question'` AND no matching answer event
- `TM_OCCURRED` in TOPIC_CREATED is TEXT — cast with `TRY_TO_TIMESTAMP(TM_OCCURRED)`
- Sort: oldest unanswered first (highest aging)

**Aging bands:**
- 🟢 < 3 days — new
- 🟡 3–7 days — needs attention
- 🟠 8–14 days — overdue
- 🔴 > 14 days — critical

---

## Output

Chat table. Full post titles, IDs, links, aging band.

```
| Post Title | Category | Age | Link |
|-----------|----------|-----|------|
| [title]   | [cat]    | [N] days | [URL] |
```

No artifact — chat table is preferred for unanswered posts so the team can act directly.

---

*Pulse capability-unanswered.md v1.0 (2026-06-03) — Extracted from Pulse v1.1. Full query spec in references/unanswered.md.*
