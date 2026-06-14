# Pulse — Heroes Pipeline Spec
# Aegis stack 2026.06.13
# Bundled (local). Load when: Heroes pipeline query spec needed (supplementary to capability-heroes.md).

---
name: heroes-pipeline
version: "1.3"
capability: heroes_pipeline
last_updated: "2026-05-12"
---

# Heroes Pipeline — Spec

## Definition

The Heroes Pipeline surfaces **non-Heroes** who score ≥ 35 on the Heroes Readiness Score — community members the Heroes program team should consider soliciting for the application. It is a **candidate list**, not an auto-nomination system.

Heroes is an application-based program with a community team–assigned role. This capability provides data to inform outreach decisions. All program decisions remain with the community team.

---

## Current Threshold

**Score threshold: ≥ 35**

At current data levels (~30 candidates at threshold 35, validated 2026-05-12).

**Threshold history:**

| Threshold | Pool size | Notes |
|---|---|---|
| 40 | ~8 | Too small (original). |
| 35 | ~16 | Selected (pre-migration data). Post-migration scores shift — ~30 candidates now. |
| 33.4 | ~46 | Hard cliff, mechanical cluster. |

---

## Heroes Readiness Score — Current Implementation

Formula and score bands: see protocol.md (canonical). Current max score is 70/100 — recency component is 0 because all CC badge records cluster on **June 26, 2025** (platform migration backfill). Date-window filtering is not meaningful until post-migration events accumulate.

`reply_milestone` = highest "Replies Authored N" badge earned. `solution_milestone` = highest "Solutions Authored N" badge earned. Communicate the 70/100 cap caveat in every output.

---

## Badge Schema — Confirmed Live

**Join key:** `USER_BADGE.ID_USER` = `COMMUNITY_USER.CAT_USER_NUMBER` (confirmed 2026-05-12)

**`TM_RECEIVED`** in `USER_BADGE` is already a TIMESTAMP — do **not** wrap in `TRY_TO_TIMESTAMP()`.

**Milestone extraction:** Badge names are formatted as "Replies Authored 100", "Solutions Authored 25", etc. Use `REGEXP_REPLACE` to extract the numeric milestone value.

---

## Data Sources

**User filter:** `BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_USER`
- `CAT_USER_NUMBER` — join key to USER_BADGE via `ID_USER`
- `CAT_USERNAME` — display name
- `CAT_ROLES` — used for exclusions and role label

**Badge tables:** `BI_CURATED.GAINSIGHT_CC.USER_BADGE` + `BI_CURATED.GAINSIGHT_CC.BADGE`
- Join: `b.ID_BADGE = ub.ID_BADGE` (confirmed 2026-05-12; NOT `b.ID = ub.ID_BADGE_ID`)
- Join to users: `USER_BADGE.ID_USER = COMMUNITY_USER.CAT_USER_NUMBER`

**Simplified role labels (for output):**

| CAT_ROLES contains | Display label |
|---|---|
| 'Verified Contributor' | Verified Contributor |
| 'Partner' | Partner |
| Anything else | Member |
| 'Employee' | Excluded from pipeline |

---

## Production Query

```sql
WITH non_heroes AS (
  SELECT CAT_USER_NUMBER, CAT_USERNAME, CAT_ROLES
  FROM BI_CURATED.STG__GAINSIGHT.STG__GAINSIGHT__COMMUNITY_USER
  WHERE CAT_ROLES NOT ILIKE '%Jamf Heroes%'
    AND CAT_ROLES NOT ILIKE '%Employee%'
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
    nh.CAT_USERNAME                                               AS username,
    COALESCE(bm.reply_milestone, 0)                              AS reply_milestone,
    COALESCE(bm.solution_milestone, 0)                           AS solution_milestone,
    CASE
      WHEN nh.CAT_ROLES ILIKE '%Verified Contributor%' THEN 'Verified Contributor'
      WHEN nh.CAT_ROLES ILIKE '%Partner%'              THEN 'Partner'
      ELSE                                                   'Member'
    END                                                          AS role_label,
    LEAST(
      LEAST(COALESCE(bm.reply_milestone, 0) / 100.0, 1.0) * 40
      + LEAST(COALESCE(bm.solution_milestone, 0) / 50.0, 1.0) * 30,
      100.0
    )                                                            AS readiness_score
  FROM non_heroes nh
  LEFT JOIN badge_milestones bm ON bm.ID_USER = nh.CAT_USER_NUMBER
)
SELECT
  username,
  ROUND(readiness_score, 1)  AS score,
  reply_milestone,
  solution_milestone,
  role_label
FROM scored
WHERE readiness_score >= 35
ORDER BY readiness_score DESC;
```

---

## Output Spec

**Format:** Chat table. Use `templates/heroes-pipeline-formatter.md`.

**Header lines (above table):**
> These members are not current Heroes and score ≥ 35 on the Heroes Readiness Score.
> Scores reflect career reply and solution milestones only. Recency component unavailable — all CC badge dates reflect the June 2025 platform migration, not actual earning dates. Max possible score is currently 70/100. Program eligibility and prior application history are not factored in.

**Table columns:**

| Column | Notes |
|---|---|
| Username | |
| Score | Numeric (max 70 currently) |
| Reply Milestone | Highest "Replies Authored N" milestone reached |
| Solution Milestone | Highest "Solutions Authored N" milestone reached |
| Role | Verified Contributor / Partner / Member |

**Footer:**
> Share with the community team to initiate application outreach.

**Empty state:**
> No candidates meet the current threshold of 35. Contact the community team to review the threshold.

---

## Output Pattern (illustrative)

At the current data level, expect roughly 30 candidates at the top of the list scoring at or near the 70/100 cap — these are members with 500+ reply milestones and 50+ solution milestones who are not yet Heroes. Render actual usernames and scores from the live query result; do not hard-code names. Example shape only:

| Username | Score | Reply Milestone | Solution Milestone | Role |
|---|---|---|---|---|
| `<handle_1>` | 70.0 | 500 | 50 | Verified Contributor |
| `<handle_2>` | 70.0 | 500 | 50 | Member |

---

## Edge Cases

| Scenario | Handling |
|---|---|
| `REGEXP_REPLACE` returns non-numeric | `TRY_CAST` returns NULL → COALESCE to 0 |
| Badge name format changes | ILIKE filters still match; REGEXP_REPLACE extracts first number found |
| Current Hero in results | Should not happen — `NOT ILIKE '%Jamf Heroes%'` applied |
| Zero results | "No candidates meet the current threshold of 35. Contact the community team to review the threshold." |
| Score > 70 | Not possible with current formula — LEAST cap at 100, but recency = 0 |

---

## Hard Rails

- Does NOT initiate contact with candidates.
- Does NOT modify Heroes status, roles, or badges.
- Does NOT make program eligibility decisions.
- Does NOT lower the threshold without explicit instruction and community team confirmation.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| v1.0 | 2026-05-11 | Initial spec. Threshold ≥ 35 from live data analysis. |
| v1.1 | 2026-05-12 | QA pass: score CTE, role simplification, hard rails. |
| v1.2 | 2026-05-12 | Live run corrections: join key → ID_USER, TM_RECEIVED no cast needed, scoring → MAX milestone from badge name (not count), recency component disabled (migration artifact). |
| v1.3 | 2026-05-12 | Badge join corrected: b.ID_BADGE = ub.ID_BADGE (not b.ID = ub.ID_BADGE_ID). Two badge CTEs merged into single badge_milestones scan. |

---

*2026.06.13 — Ported from the Pulse archive package; scrubbed, CalVer. The "Notable Candidates" section (11 real Jamf Nation usernames + scores) was removed for the public repo and replaced with a generic illustrative output pattern.*
