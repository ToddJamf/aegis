# Pulse — Dormant Account List Spec
# Aegis stack 2026.06.13
# Bundled (local). Load when: dormant query spec needed (supplementary to capability-dormant.md).

---
name: dormant
version: "2.1"
capability: dormant
last_updated: "2026-05-12"
---

# Dormant Account List — Spec

## Definition

A **dormant account** is a company-level account that:
1. Has at least one historical engagement event (ever active)
2. Has not had any engagement event in the last 90+ days
3. Is NOT managed by the Digital Team

**Dark accounts** (never active) are NOT dormant. Different motion, different output. Do not mix them.

---

## Tiers

| Tier | Days Inactive | Color | Priority |
|---|---|---|---|
| Amber | 90–179 days | `#E08A00` | Recent lapse — re-engagement priority. Display **above** Red. |
| Red | 180+ days | `#C41E3A` | Extended lapse |

---

## Data Source

**Primary table:** `BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_ENGAGEMENT_OVERVIEW`

Relevant columns:
- `CAT_COMPANY_NAME` — company name (join key to ARR)
- `CAT_CSM_NAME` — CSM name (exclusion filter + user filter)
- `N_PAGE_VIEWS` — page view count
- `TM_LAST_ACTIVITY` — last engagement timestamp (TIMESTAMP_NTZ)
- `N_LAST_SEEN_DAYS_AGO` — days since last activity (NUMBER)

**ARR join table:** `BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMPANY_ALL_COLUMNS`

Relevant columns:
- `CAT_NAME` — company name (join key)
- `AMT_ARR_USD` — ARR in USD
- `ID_ARR_BAND` — ARR band GSID (map to label using GSID table in SKILL.md)

**Join path:**
```sql
LEFT JOIN BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMPANY_ALL_COLUMNS co
  ON co.CAT_NAME = e.CAT_COMPANY_NAME
```

---

## Production Query

```sql
WITH dormant_accounts AS (
  SELECT
    e.CAT_COMPANY_NAME                                    AS company,
    e.CAT_CSM_NAME                                        AS csm,
    e.TM_LAST_ACTIVITY                                    AS last_activity,
    e.N_LAST_SEEN_DAYS_AGO                                AS days_inactive,
    e.N_PAGE_VIEWS                                        AS page_views,
    CASE
      WHEN e.N_LAST_SEEN_DAYS_AGO BETWEEN 90 AND 179 THEN 'Amber'
      WHEN e.N_LAST_SEEN_DAYS_AGO >= 180              THEN 'Red'
    END                                                   AS tier,
    CASE
      WHEN e.N_LAST_SEEN_DAYS_AGO BETWEEN 90 AND 179 THEN 1
      WHEN e.N_LAST_SEEN_DAYS_AGO >= 180              THEN 2
    END                                                   AS tier_sort,
    CASE co.ID_ARR_BAND
      WHEN '1I00ONSM92NZS4XF7Q5CLYL0M3D6SU9AMDR9' THEN '$10k or less'
      WHEN '1I00ONSM92NZS4XF7QIRWZNOF3IFFF6YNTTV' THEN '$10k – $50k'
      WHEN '1I00ONSM92NZS4XF7Q5DS0FHZUXUDW2KCYB7' THEN '$50k – $100k'
      WHEN '1I00ONSM92NZS4XF7QBPF2L1XWTVY15Y2H2Y' THEN '$100k – $250k'
      WHEN '1I00ONSM92NZS4XF7Q8LN2ZJZM9AHDVBX6RR' THEN '$250k or more'
      ELSE 'Unknown'
    END                                                   AS arr_band
  FROM BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_ENGAGEMENT_OVERVIEW e
  LEFT JOIN BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMPANY_ALL_COLUMNS co
    ON co.CAT_NAME = e.CAT_COMPANY_NAME
  WHERE e.CAT_CSM_NAME != 'Jamf Digital Team'
    AND e.TM_LAST_ACTIVITY IS NOT NULL
    AND e.N_LAST_SEEN_DAYS_AGO IS NOT NULL
    AND e.N_LAST_SEEN_DAYS_AGO >= 90
)
SELECT
  company,
  csm,
  arr_band,
  last_activity,
  days_inactive,
  page_views,
  tier
FROM dormant_accounts
ORDER BY tier_sort ASC, days_inactive DESC;
```

---

## Output Spec

**Format:** HTML artifact always. Use `templates/dormant-template.html`.

**Columns:**

| Column | Source | Notes |
|---|---|---|
| Company | `CAT_COMPANY_NAME` | |
| CSM | `CAT_CSM_NAME` | |
| ARR Band | GSID map | Show "Unknown¹" for null/unmatched |
| Last Activity | `TM_LAST_ACTIVITY` | Format: YYYY-MM-DD |
| Days Inactive | `N_LAST_SEEN_DAYS_AGO` | |
| Page Views | `N_PAGE_VIEWS` | |
| Tier | Derived | Pill badge — Amber or Red |

**Sort order:** Amber tier above Red. Within each tier, sort by days inactive descending.

**Filters:** ARR Band dropdown · CSM name dropdown

**Footer lines:**
- `¹ Unknown ARR = no ARR data on file for this account in Gainsight.`
- `Contact CS Ops for full export.`

---

## Edge Cases

| Scenario | Handling |
|---|---|
| `N_LAST_SEEN_DAYS_AGO` IS NULL | Exclude from results. NULL means no activity data — row should not appear in dormant list. |
| `TM_LAST_ACTIVITY` IS NULL | Exclude from results. Filter applied in WHERE clause. |
| `AMT_ARR_USD` = 0.00 | Include in results. Zero-ARR may indicate churned or inactive account — community team decides whether to engage. |
| No ARR join match | LEFT JOIN preserves account. ARR Band shows "Unknown¹". Include in results. |
| CSM filter active, no results | Render filtered empty state: "No dormant accounts found for '[CSM name]'. CSM names must match Gainsight exactly — try the full name as it appears in your book." |
| No results (unfiltered) | Render: "No dormant accounts found." |
| `ID_ARR_BAND` has unlisted GSID | Falls to ELSE → "Unknown¹" — handled by CASE. |

---

## Hard Rails

- Does NOT list dark accounts (never active). Dark accounts are a separate motion.
- Does NOT include Digital Team accounts (`CAT_CSM_NAME != 'Jamf Digital Team'` always applied).
- Does NOT auto-export or email results. Footer directs user to CS Ops for full export.
- Does NOT make re-engagement recommendations — surfaces data only.
- Does NOT filter by zero-ARR automatically — that is a program decision, not a data decision.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| v1.0 | 2026-05-11 | Initial build. Basic dormant list, no ARR column. |
| v2.0 | 2026-05-11 | ARR join added. Two-tier display. CS Ops footer. HTML artifact. |
| v2.1 | 2026-05-12 | QA pass: NULL handling explicit, edge cases table, hard rails section, filtered empty state, Unknown ARR footnote. |

---

*2026.06.13 — Ported from the Pulse archive package; scrubbed, CalVer.*
