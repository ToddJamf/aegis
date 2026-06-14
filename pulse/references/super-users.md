# Pulse — Heroes Tracker Spec
# Aegis stack 2026.06.13
# Bundled (local). Load when: Heroes tracker query spec needed (supplementary to capability-heroes.md).

---
name: super-users
version: "1.3"
capability: super_users
last_updated: "2026-05-12"
---

# Heroes Tracker — Spec

## Definition

The Heroes Tracker surfaces all current Jamf Heroes — users with `CAT_ROLES ILIKE '%Jamf Heroes%'` — alongside their Heroes Readiness Score. It is an **operational tracker**, not a nomination pipeline. It answers: "Are our current Heroes staying engaged?"

Current Heroes count: ~132 (as of 2026-05-11).

---

## Heroes Readiness Score

Formula and score bands: see protocol.md (canonical). Current max score is 70/100 — recency component disabled (CC badge dates are June 2025 migration artifacts, not actual earning dates).

**Data coverage caveat — always surface in output:** Scores reflect career badge milestones only. Tenured Heroes with pre-Aug 2024 activity may score lower than their actual contribution level.

---

## Data Sources

**Heroes roster:** `BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_USER`
- `CAT_USER_NUMBER` — join key to badge tables
- `CAT_ROLES` — filter: `ILIKE '%Jamf Heroes%'`
- `CAT_USERNAME` — display name

**Badge activity:** `BI_CURATED.GAINSIGHT_CC.USER_BADGE`
- `ID_USER` — join key to COMMUNITY_USER.CAT_USER_NUMBER (confirmed 2026-05-12)
- `ID_BADGE` — badge join key. Join to BADGE on `b.ID_BADGE = ub.ID_BADGE` (confirmed 2026-05-12; NOT `ID_BADGE_ID`)
- `TM_RECEIVED` — already TIMESTAMP, do NOT wrap in TRY_TO_TIMESTAMP()

**Badge metadata:** `BI_CURATED.GAINSIGHT_CC.BADGE`
- `ID_BADGE` — primary key (NOT `ID` — confirmed 2026-05-12)
- `CAT_NAME` — badge name, e.g. "Replies Authored 500", "Solutions Authored 25"

**Milestone extraction:** Use `MAX(TRY_CAST(REGEXP_REPLACE(b.CAT_NAME, '[^0-9]', '') AS INTEGER))` with conditional MAX to fetch both milestones in a single badge scan.

**Data caveat — recency unavailable:** All CC badge records cluster around June 26, 2025 (platform migration backfill). Date-window filtering (365d, 90d) returns near-zero meaningful signal. Recency component of the score is disabled until post-migration badge events accumulate. Current max score is 70/100.

---

## Production Query

```sql
WITH heroes AS (
  SELECT CAT_USER_NUMBER, CAT_USERNAME
  FROM BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_USER
  WHERE CAT_ROLES ILIKE '%Jamf Heroes%'
),
badge_milestones AS (
  -- Single scan: fetches reply + solution milestones in one pass
  SELECT
    ub.ID_USER,
    MAX(CASE WHEN b.CAT_NAME ILIKE '%replies authored%'
        THEN TRY_CAST(REGEXP_REPLACE(b.CAT_NAME, '[^0-9]', '') AS INTEGER) END) AS reply_milestone,
    MAX(CASE WHEN b.CAT_NAME ILIKE '%solutions authored%'
        THEN TRY_CAST(REGEXP_REPLACE(b.CAT_NAME, '[^0-9]', '') AS INTEGER) END) AS solution_milestone
  FROM BI_CURATED.GAINSIGHT_CC.USER_BADGE ub
  JOIN BI_CURATED.GAINSIGHT_CC.BADGE b ON b.ID_BADGE = ub.ID_BADGE
  WHERE b.CAT_NAME ILIKE '%replies authored%'
     OR b.CAT_NAME ILIKE '%solutions authored%'
  GROUP BY ub.ID_USER
),
scored AS (
  SELECT
    h.CAT_USERNAME                                               AS username,
    COALESCE(bm.reply_milestone, 0)                             AS reply_milestone,
    COALESCE(bm.solution_milestone, 0)                          AS solution_milestone,
    LEAST(
      LEAST(COALESCE(bm.reply_milestone, 0) / 100.0, 1.0) * 40
      + LEAST(COALESCE(bm.solution_milestone, 0) / 50.0, 1.0) * 30,
      100.0
    )                                                            AS readiness_score
  FROM heroes h
  LEFT JOIN badge_milestones bm ON bm.ID_USER = h.CAT_USER_NUMBER
)
SELECT
  username,
  ROUND(readiness_score, 1)  AS score,
  reply_milestone,
  solution_milestone,
  CASE
    WHEN readiness_score >= 70 THEN 'Active Hero'
    WHEN readiness_score >= 40 THEN 'Fading'
    WHEN readiness_score >= 10 THEN 'At Risk'
    ELSE                            'Inactive'
  END                        AS band
FROM scored
ORDER BY
  CASE band
    WHEN 'Active Hero' THEN 1
    WHEN 'Fading'      THEN 2
    WHEN 'At Risk'     THEN 3
    ELSE                    4
  END,
  readiness_score DESC;
```

---

## Output Spec

**Format:** HTML artifact always. Use `templates/super-users-template.html`.

**Summary strip (top of artifact):**

| Tile | Value |
|---|---|
| Total Heroes | COUNT of all rows |
| Active | COUNT where band = 'Active Hero' |
| Fading | COUNT where band = 'Fading' |
| At Risk | COUNT where band = 'At Risk' |
| Inactive | COUNT where band = 'Inactive' |

**Caveat line (below summary strip):**
> Scores reflect career reply and solution milestones only. Recency component unavailable — CC badge dates reflect the June 2025 platform migration, not actual earning dates. Max possible score is currently 70/100.

Style: 0.85rem, Haiti at 60% opacity. Not an error — informational only.

**Table columns:**

| Column | Notes |
|---|---|
| Username | Display name |
| Score | Progress bar (0–70 effective max), colored by band. Numeric value at bar end. |
| Reply Milestone | Highest "Replies Authored N" milestone reached |
| Solution Milestone | Highest "Solutions Authored N" milestone reached |
| Band | Pill badge |

**Sort:** Active Hero → Fading → At Risk → Inactive. Within band, score descending.

**Band filter:** Dropdown to filter by band. Inactive rows **collapsed by default** — show toggle "Show X inactive Heroes ▼". Expands on click.

**NULL last_badge_earned:** Render "No CC activity recorded" in the Last Badge cell. Do not show blank or error.

---

## Edge Cases

| Scenario | Handling |
|---|---|
| Hero has no badge activity in CC tables | All metrics = 0, score = 0, band = 'Inactive' |
| `replies_90d` > `replies_365d` | Data anomaly — LEAST cap at 100.0 prevents score > 100 |
| `last_badge_earned` IS NULL | Show "No CC activity recorded" |
| `solution_milestone` IS NULL | COALESCE to 0 |
| Recency ratio NULL (0 replies both windows) | NULLIF prevents divide by zero — recency component = 0 |

---

## Hard Rails

- Does NOT recommend Heroes for removal or non-renewal.
- Does NOT modify user roles, badges, or program status.
- Does NOT make program decisions (renewal, removal, recognition).
- Does NOT surface individual Heroes' data outside the team viewer.
- Does NOT compare Heroes to each other for competitive ranking — scores are operational signals, not rankings.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| v1.0 | 2026-05-11 | Heroes tracker spec + formula derived from live data analysis. 132 Heroes confirmed. Score bands validated against known Heroes. |
| v1.1 | 2026-05-12 | QA pass: LEAST cap, band filter, NULL last_badge display, hard rails. |
| v1.2 | 2026-05-12 | Live run corrections: join key → ID_USER, TM_RECEIVED no cast, scoring → MAX milestone from badge name, recency disabled (migration artifact), columns updated to reply_milestone/solution_milestone. |
| v1.3 | 2026-05-12 | Badge join corrected: b.ID_BADGE = ub.ID_BADGE (not b.ID = ub.ID_BADGE_ID). Two badge CTEs merged into single badge_milestones scan. |

---

*2026.06.13 — Ported from the Pulse archive package; scrubbed, CalVer.*
