# Pulse — Weekly KPI Report Render Spec
# Aegis stack 2026.06.13
# Bundled (local). Load when: dashboard / KPI report render spec needed.

---
name: dashboard
version: "1.0"
last_updated: "2026-05-12"
---

# Pulse — Weekly KPI Report Spec

Governs the `dashboard` capability. Triggered by: "community dashboard", "KPI report", "weekly KPI", "how is Jamf Nation doing", "run weekly KPI report".

Output: HTML rendered inline, then converted to PDF via `references/pdf-template.html`.

---

## Report Sections (in order)

### 1. Header
- Title: **Jamf Nation — Weekly Community KPI Report**
- Subtitle: `Rolling 30 days ending [CURRENT_DATE]`
- Short AI disclaimer (header short-form) per `shared/output-discipline.md`

### 2. Headline KPI Cards (top row)

Four cards, always present:

| Card | Metric | Source |
|---|---|---|
| Active Accounts | Count of accounts with ≥1 engagement event (30d) | CC event tables + account join |
| Engagement Events | Total replies + likes (30d) | CONTENT_TOPIC_REPLIED + REPLY_LIKED + TOPIC_LIKED |
| Topics Created | New topics posted (30d) | CONTENT_TOPIC_CREATED |
| New Registrations | Users registered (30d) | STG__GAINSIGHT__COMMUNITY_USER |

Card format: large number, metric label, delta vs. prior 30-day period (▲/▼/—).

### 3. Unanswered Questions Summary

Three-band summary table:

| Band | Count | Age |
|---|---|---|
| 🟡 Amber | N | 7–14 days |
| 🔴 Red | N | 15–30 days |
| 🔴 Critical | N | 30+ days |

- **Spam bucket:** Count of posts in "Mark as spam" category surfaced separately below the table.
  - Label: `⚠️ Spam bucket: N posts in "Mark as spam" — flagged for takedown review`
- Amber full list is NOT included in the dashboard report. Dashboard shows counts only. Run the `unanswered` capability for the full queue.

### 4. Engagement Breakdown (30d)

Simple table:

| Metric | Count |
|---|---|
| Replies | N |
| Topic likes | N |
| Reply likes | N |
| **Total events** | **N** |

### 5. Reply Rate

- Metric: `X% of topics received at least one reply (30d)`
- Context line: `N of M topics created in the last 30 days have at least one reply.`

### 6. Footer

Full AI disclaimer per `shared/output-discipline.md` (substantive HTML output → full footer).

---

## Queries

### Active Accounts (30d)

```sql
SELECT COUNT(*) AS active_accounts_30d
FROM BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_ENGAGEMENT_OVERVIEW
WHERE TM_LAST_ACTIVITY >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND CAT_CSM_NAME != 'Jamf Digital Team';
```

**Confirmed field:** `TM_LAST_ACTIVITY` (timestamp_ntz). Confirmed 2026-05-12.

**Delta limitation (confirmed 2026-05-12):** Two approaches tested:
- `TM_LAST_ACTIVITY` proxy → current vs. prior. Prior is undercounted (active accounts show in current window only, not prior). Not comparable.
- CC event tables via `ID_USER → CAT_USER_NUMBER → CAT_COMPANY_CUSTOM_NAME` → massively undercounts — `CAT_COMPANY_CUSTOM_NAME` is sparsely populated in COMMUNITY_USER; most users don't have it filled in.

**Current guidance:** Use the `TM_LAST_ACTIVITY` count as the best available absolute figure. **Skip delta until a reliable user→company join is confirmed.** Flag in output as `delta methodology pending`. Correct join likely requires `ID_COMPANY` on COMMUNITY_USER joined to ENGAGEMENT_OVERVIEW — investigate with data team.

### Engagement Events (30d)

```sql
SELECT
  SUM(replies) + SUM(reply_likes) + SUM(topic_likes) AS total_events,
  SUM(replies)      AS replies,
  SUM(reply_likes)  AS reply_likes,
  SUM(topic_likes)  AS topic_likes
FROM (
  SELECT COUNT(*) AS replies, 0 AS reply_likes, 0 AS topic_likes
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_REPLIED
  WHERE TM_OCCURRED >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  UNION ALL
  SELECT 0, COUNT(*), 0
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_REPLY_LIKED
  WHERE TM_OCCURED >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  UNION ALL
  SELECT 0, 0, COUNT(*)
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_LIKED
  WHERE TM_OCCURED >= DATEADD('day', -30, CURRENT_TIMESTAMP())
) t;
```

### Topics Created (30d)

```sql
SELECT COUNT(*) AS topics_created_30d
FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_CREATED
WHERE TRY_TO_TIMESTAMP(TM_OCCURRED) >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND CAT_CATEGORY_NAME NOT IN ('Old Posts', 'Archived Beta Programs', 'Job Board', 'Partners', 'User Group Leaders');
```

### New Registrations (30d)

```sql
SELECT COUNT(*) AS new_registrations_30d
FROM BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_USER
WHERE TM_REGISTERED_ON >= DATEADD('day', -30, CURRENT_TIMESTAMP());
```

**Confirmed field:** `TM_REGISTERED_ON` (timestamp_ntz). Confirmed 2026-05-12.

### Unanswered Questions — Band Counts + Spam Bucket

```sql
WITH questions AS (
  SELECT
    ID_TOPIC_PUBLIC,
    CAT_CATEGORY_NAME,
    DATEDIFF('day', TRY_TO_TIMESTAMP(TM_OCCURRED), CURRENT_TIMESTAMP()) AS age_days
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_CREATED
  WHERE CAT_CONTENT_TYPE = 'question'
    AND TRY_TO_TIMESTAMP(TM_OCCURRED) IS NOT NULL
    AND DATEDIFF('day', TRY_TO_TIMESTAMP(TM_OCCURRED), CURRENT_TIMESTAMP()) >= 7
),
answered AS (
  SELECT DISTINCT ID_TOPIC_PUBLIC
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_QUESTION_ANSWERED
  WHERE ID_TOPIC_PUBLIC IS NOT NULL
),
unanswered AS (
  SELECT
    q.ID_TOPIC_PUBLIC,
    q.CAT_CATEGORY_NAME,
    q.age_days,
    CASE
      WHEN q.age_days BETWEEN 7  AND 14 THEN 'Amber'
      WHEN q.age_days BETWEEN 15 AND 30 THEN 'Red'
      WHEN q.age_days > 30              THEN 'Critical'
    END AS aging_band,
    CASE WHEN q.CAT_CATEGORY_NAME = 'Mark as spam' THEN TRUE ELSE FALSE END AS spam_bucket
  FROM questions q
  LEFT JOIN answered a ON a.ID_TOPIC_PUBLIC = q.ID_TOPIC_PUBLIC
  WHERE a.ID_TOPIC_PUBLIC IS NULL
    AND q.CAT_CATEGORY_NAME NOT IN ('Old Posts', 'Job Board', 'Partners', 'User Group Leaders')
)
SELECT
  aging_band,
  spam_bucket,
  COUNT(*) AS cnt
FROM unanswered
GROUP BY aging_band, spam_bucket
ORDER BY
  spam_bucket,
  CASE aging_band WHEN 'Amber' THEN 1 WHEN 'Red' THEN 2 ELSE 3 END;
```

Post-process: rows where `spam_bucket = TRUE` sum to the spam bucket count. Rows where `spam_bucket = FALSE` and `aging_band IS NOT NULL` sum to the standard band counts.

### Reply Rate (30d)

```sql
WITH topics AS (
  SELECT ID_TOPIC_PUBLIC
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_CREATED
  WHERE TRY_TO_TIMESTAMP(TM_OCCURRED) >= DATEADD('day', -30, CURRENT_TIMESTAMP())
    AND CAT_CATEGORY_NAME NOT IN ('Old Posts', 'Archived Beta Programs', 'Job Board', 'Partners', 'User Group Leaders')
),
with_replies AS (
  SELECT DISTINCT t.ID_TOPIC_PUBLIC
  FROM topics t
  INNER JOIN BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_REPLIED r
    ON r.ID_TOPIC_PUBLIC = t.ID_TOPIC_PUBLIC
)
SELECT
  COUNT(t.ID_TOPIC_PUBLIC) AS total_topics,
  COUNT(wr.ID_TOPIC_PUBLIC) AS topics_with_replies,
  ROUND(COUNT(wr.ID_TOPIC_PUBLIC) * 100.0 / NULLIF(COUNT(t.ID_TOPIC_PUBLIC), 0), 1) AS reply_rate_pct
FROM topics t
LEFT JOIN with_replies wr ON wr.ID_TOPIC_PUBLIC = t.ID_TOPIC_PUBLIC;
```

---

## Delta Calculation (MoM)

For each headline KPI, run the same query for the prior 30-day window (`DATEADD('day', -60, ...)` to `DATEADD('day', -30, ...)`) and compute:
- `delta = current - prior`
- `delta_pct = ROUND((current - prior) * 100.0 / NULLIF(prior, 0), 1)`
- Display as `▲ N (+X%)` or `▼ N (-X%)` or `— (no change)`

---

## Rendering

Pass all query results to `references/pdf-template.html`. The template handles layout, Jamf brand styling, KPI cards, tables, and PDF generation.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| v1.0 | 2026-05-12 | Initial spec. 30-day rolling window. Category exclusions applied. Spam bucket surfaced separately. |

---

*2026.06.13 — Ported from the Pulse archive package; scrubbed, CalVer. Sample figures removed from the delta-limitation note (illustrative counts genericized). AI-disclaimer references repointed from the superseded references/ai-footer.md to shared/output-discipline.md per the Aegis shared layer. SQL, KPI logic, and category exclusions intact.*
