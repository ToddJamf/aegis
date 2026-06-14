# Pulse — Locked KPI Definitions
# Aegis stack 2026.06.13
# Bundled (local). Load when: any query needs metric definitions, exclusion rules, or data-quality fixes.

---
name: kpi-definitions
version: "1.0"
last_updated: "2026-05-12"
---

# Pulse — Locked KPI Definitions

All metrics in the Weekly KPI report and ops dashboard are derived from these definitions. Do not deviate without updating this file and bumping the version.

---

## Core Metric Definitions

### Active Accounts (30d)
**Definition:** Count of distinct customer accounts with at least one engagement event in the rolling 30-day window.
**Engagement event:** A LIKE or a REPLY. Page views are NOT engagement events.
**Exclusion:** `CAT_CSM_NAME != 'Jamf Digital Team'`
**Source:** `STG__GAINSIGHT__COMMUNITY_ENGAGEMENT_OVERVIEW` joined to CC event tables.
**Window:** Rolling 30 calendar days ending at time of query.

### Engagement Events (30d)
**Definition:** Total count of LIKE and REPLY events in the rolling 30-day window.
**Components:**
- Replies: rows in `CONTENT_TOPIC_REPLIED` where `TM_OCCURRED` within window
- Reply likes: rows in `CONTENT_REPLY_LIKED` where `TM_OCCURED` within window (one R — schema typo)
- Topic likes: rows in `CONTENT_TOPIC_LIKED` where `TM_OCCURED` within window (one R — schema typo)
**Exclusion:** Digital Team accounts excluded at account join level.
**Window:** Rolling 30 days.

### Topics Created (30d)
**Definition:** Count of new topics posted in the rolling 30-day window.
**Source:** `CONTENT_TOPIC_CREATED` — `TRY_TO_TIMESTAMP(TM_OCCURRED)` within window.
**Note:** `TM_OCCURRED` is TEXT in this table — always cast.
**Scope:** All categories except excluded list (see Category Exclusions below).

### Reply Rate (30d)
**Definition:** Percentage of topics created in the last 30 days that have received at least one reply.
**Formula:** `COUNT(topics with ≥1 reply) / COUNT(total topics) * 100`
**Reply source:** `CONTENT_TOPIC_REPLIED` — count rows per `ID_TOPIC_PUBLIC`.

### Unanswered Questions (open count)
**Definition:** Topics where `CAT_CONTENT_TYPE = 'question'`, no row in `CONTENT_QUESTION_ANSWERED`, and at least 7 days old.
**Aging bands:**
- Amber: 7–14 days
- Red: 15–30 days
- Critical: 30+ days
**Full spec:** `references/unanswered.md`

### New Registered Users (30d)
**Definition:** Count of users registered in the rolling 30-day window.
**Source:** `STG__GAINSIGHT__COMMUNITY_USER` — `DT_REGISTERED` within window.
**Scope:** All users (not limited to customer-linked accounts).

### Registered Users — Customer-Linked (point-in-time)
**Definition:** Users linked to a Gainsight customer account. ~35K population at time of last count.
**IMPORTANT:** Do NOT mix with total Jamf Nation population (~162K). These are two distinct figures and must never be presented as interchangeable.

---

## Category Exclusions

The following categories are excluded from all general topic counts and queues. Confirmed by the community team, 2026-05-12.

| Category | Parent | Reason | Treatment in unanswered queue |
|---|---|---|---|
| Mark as spam | Catch All | Spam posts retained for ML/spam-detection training (Khoros migration artifact) | ⚠️ NOT in CC data — see note below |
| Old Posts | Catch All | Legacy archived discussions — not customer-facing | Exclude entirely |
| Archived Beta Programs | Catch All | Archived beta content — not customer-facing | Exclude entirely |
| Job Board | — | Job listings, not support content | Exclude entirely |
| Partners | — | Permission-gated (requires partner role) | Exclude from general queue |
| User Group Leaders | — | Permission-gated (requires UG Leader custom role granted manually) | Exclude from general queue |

**⚠️ "Mark as spam" CC data gap (confirmed 2026-05-12):** This category returns 0 rows in `GAINSIGHT_CC.CONTENT_TOPIC_CREATED`. CC tables cover August 2024 onward — the spam posts likely predate this window or live in STG tables. Spam bucket will always return 0 via CC query. **Needs the community team to confirm category name and table.** Until resolved, suppress the spam bucket section from output (don't show "0" — misleading).

**"Archived Beta Programs" vs "Old Posts":** Confirmed as two distinct categories in CC data. Both excluded. The community team described these together as "Old Posts" but they are separate in the system.

**Gated categories rationale:** Partners and User Group Leaders are not customer-visible without special permissions. Surfacing them in the general queue would flag posts most CSMs cannot access or action.

---

## Time Window Rules

**Rolling 30 days:** All KPIs use rolling 30 calendar days ending at `CURRENT_TIMESTAMP()` unless the specific metric definition states otherwise.

**Partial month flag:** Any in-progress calendar month gets a ★ flag on chart labels. Do not extrapolate partial-month data to full-month projections.

**CC table coverage:** CC event tables (`GAINSIGHT_CC` schema) start August 2024. Pre-Aug 2024 trend history requires `STG__GAINSIGHT` tables. Trend queries spanning both periods must union both sources.

---

## Exclusion Rules (Always Apply)

```sql
WHERE CAT_CSM_NAME != 'Jamf Digital Team'
```

Apply at the account join level, not as a post-filter. This removes all internal Jamf Digital Team activity from engagement metrics.

---

## Data Quality Notes

| Issue | Correct approach |
|---|---|
| `TM_OCCURED` one R on `REPLY_LIKED` and `TOPIC_LIKED` | Use `TM_OCCURED` — schema typo, do not correct |
| `TM_OCCURRED` two Rs on `TOPIC_REPLIED` | Use `TM_OCCURRED` |
| `TM_OCCURRED` TEXT on `TOPIC_CREATED` and `QUESTION_ANSWERED` | `TRY_TO_TIMESTAMP(TM_OCCURRED)` before any date function |
| No `N_REPLIES` column in `CONTENT_TOPIC_CREATED` | Count rows in `CONTENT_TOPIC_REPLIED` instead |
| CC badge dates cluster June 26, 2025 | Migration artifact — Heroes recency scoring disabled until new events accumulate |
| CC schema has no STG__ prefix | `BI_CURATED.GAINSIGHT_CC` — not `BI_CURATED.STG__GAINSIGHT_CC` |
| Non-CC table naming | `STG__GAINSIGHT__TABLENAME` double-underscore |

---

## Version History

| Version | Date | Notes |
|---|---|---|
| v1.0 | 2026-05-12 | Initial file. Derived from SKILL.md definitions. Category exclusions confirmed by the community team 2026-05-12. |

---

*2026.06.13 — Ported from the Pulse archive package; scrubbed, CalVer. Employee name (community team contact) replaced with "the community team."*
