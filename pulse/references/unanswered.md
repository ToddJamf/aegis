# Pulse — Unanswered Posts Queue Spec
# Aegis stack 2026.06.13
# Bundled (local). Load when: unanswered posts query spec needed (supplementary to capability-unanswered.md).

---
name: unanswered
version: "1.3"
capability: unanswered
last_updated: "2026-05-12"
---

# Unanswered Posts Queue — Spec

## Definition

An **unanswered post** is a community topic that:
1. Has `CAT_CONTENT_TYPE = 'question'`
2. Has NO row in `CONTENT_QUESTION_ANSWERED` matching its `ID_TOPIC_PUBLIC`
3. Is at least 7 days old

---

## Output Design

**Amber (7–14 days):** Full list, sorted newest first (age_days ASC). These are actionable today.

**Red (15–30 days) and Critical (30d+):** Bucket counts only. Available on request if the community team wants to drill in.

This keeps the output focused. Don't dump 1,200+ Critical posts into chat — surface them as a count and offer to pull them if asked.

---

## Spam Flagging

Posts suspected as spam are flagged with ⚠️ in the output. The flag is heuristic — community team reviews and acts. Claude does not delete or hide flagged posts.

**Spam heuristic:** A post is flagged when ALL three conditions are true:
1. Zero replies
2. Category is 'General Discussions'
3. Title contains none of the Jamf/IT keyword list (see query)

**Known false positive pattern:** Generic tech phrases without Jamf keywords (e.g. "Media lookup failed to find the requested identifier") can trip the flag. Always treat ⚠️ as "worth a look" not "confirmed spam."

---

## Data Sources

**Topics table:** `BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_CREATED`
- `ID_TOPIC_PUBLIC` — topic ID, join key
- `CAT_TITLE` — full topic title
- `CAT_CATEGORY_NAME` — community board
- `CAT_CONTENT_TYPE` — filter to `'question'`
- `TM_OCCURRED` — creation timestamp (TEXT — must cast)

Note: `N_REPLIES` does not exist in this table. Reply counts come from `CONTENT_TOPIC_REPLIED`.

**Reply count table:** `BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_REPLIED`
- Count rows per `ID_TOPIC_PUBLIC`

**Answered events table:** `BI_CURATED.GAINSIGHT_CC.CONTENT_QUESTION_ANSWERED`
- `ID_TOPIC_PUBLIC` — join key
- `TM_OCCURRED` — answer timestamp (TEXT — must cast)

**Schema:** `BI_CURATED.GAINSIGHT_CC` (no STG__ prefix — confirmed live 2026-05-12)

---

## Production Query

```sql
WITH questions AS (
  SELECT
    ID_TOPIC_PUBLIC,
    CAT_TITLE,
    CAT_CATEGORY_NAME,
    DATEDIFF('day', TRY_TO_TIMESTAMP(TM_OCCURRED), CURRENT_TIMESTAMP()) AS age_days
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_CREATED
  WHERE CAT_CONTENT_TYPE = 'question'
    AND TRY_TO_TIMESTAMP(TM_OCCURRED) IS NOT NULL
    AND DATEDIFF('day', TRY_TO_TIMESTAMP(TM_OCCURRED), CURRENT_TIMESTAMP()) >= 7
),
reply_counts AS (
  SELECT ID_TOPIC_PUBLIC, COUNT(*) AS replies
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_TOPIC_REPLIED
  GROUP BY ID_TOPIC_PUBLIC
),
answered AS (
  SELECT DISTINCT ID_TOPIC_PUBLIC
  FROM BI_CURATED.GAINSIGHT_CC.CONTENT_QUESTION_ANSWERED
  WHERE ID_TOPIC_PUBLIC IS NOT NULL
),
unanswered AS (
  SELECT
    q.ID_TOPIC_PUBLIC,
    q.CAT_TITLE,
    q.CAT_CATEGORY_NAME,
    q.age_days,
    COALESCE(r.replies, 0)                                AS replies,
    CASE
      WHEN q.age_days BETWEEN 7  AND 14 THEN 'Amber'
      WHEN q.age_days BETWEEN 15 AND 30 THEN 'Red'
      WHEN q.age_days > 30              THEN 'Critical'
    END                                                   AS aging_band,
    CASE
      WHEN COALESCE(r.replies, 0) = 0
        AND q.CAT_CATEGORY_NAME = 'General Discussions'
        AND NOT (
          LOWER(q.CAT_TITLE) LIKE ANY (
            '%jamf%','%mac%','%ios%','%ipad%','%iphone%','%mdm%','%device%',
            '%apple%','%enroll%','%deploy%','%manage%','%profile%','%policy%',
            '%script%','%update%','%install%','%network%','%wifi%','%vpn%',
            '%security%','%certificate%','%api%','%app %','%computer%',
            '%mobile%','%patch%','%config%','%self service%','%beta%',
            '%password%','%user%','%admin%','%token%','%keychain%','%azure%',
            '%active directory%','%ldap%','%sso%','%okta%','%microsoft%',
            '%google%','%chrome%','%windows%','%outlook%','%teams%',
            '%airdrop%','%bluetooth%','%usb%','%printer%'
          )
        )
      THEN TRUE ELSE FALSE
    END                                                   AS spam_flag
  FROM questions q
  LEFT JOIN reply_counts r ON r.ID_TOPIC_PUBLIC = q.ID_TOPIC_PUBLIC
  LEFT JOIN answered a     ON a.ID_TOPIC_PUBLIC = q.ID_TOPIC_PUBLIC
  WHERE a.ID_TOPIC_PUBLIC IS NULL
)
SELECT
  ID_TOPIC_PUBLIC,
  CAT_TITLE,
  CAT_CATEGORY_NAME,
  age_days,
  replies,
  aging_band,
  spam_flag
FROM unanswered
ORDER BY
  CASE aging_band WHEN 'Amber' THEN 1 WHEN 'Red' THEN 2 ELSE 3 END,
  age_days ASC;
```

---

## Output Spec

**Format:** Chat table. No HTML artifact.

**Opening line:**
```
X unanswered questions — Amber: Y · Red: Z · Critical: W
```

**Bucket summary table (always show first):**

| Band | Count | Age Range |
|---|---|---|
| 🟡 Amber | Y | 7–14 days — shown below |
| 🔴 Red | Z | 15–30 days — available on request |
| 🔴 Critical | W | 30+ days — available on request |

**Amber list (full, newest first):**

Header: `🟡 Amber — Act Now (7–14 days) · Y posts · newest first`

| Topic | Category | Age | Replies |
|---|---|---|---|
| [full title] ⚠️ if spam_flag | category | Xd | N |

- Full title — do not truncate
- Append ` ⚠️` to title if `spam_flag = true`
- Sort: age_days ASC (7-day posts first)
- Include topic ID in output if URL format is confirmed; otherwise omit

**Footer:**
```
Red (Z) and Critical (W) available on request.
⚠️ = suspected spam (zero replies, General Discussions, no Jamf/IT keywords) — review before actioning.
```

**Empty state:** "No unanswered questions older than 7 days. 🎉"

**If Red or Critical requested:** Run the same query filtered to that band, same spam flagging, oldest first (age_days DESC — most overdue at top).

---

## Volume Note

At last run (2026-05-12): 1,259 total unanswered. The Critical bucket (1,191) is dominated by:
- Migration/test content from June 2025 platform launch (titles like "asdf", "Testing", "Who let the dogs out?")
- Spam wave ~July 4–5, 2025 (casino, betting, jewelry, real estate listings)
- Some genuine older tech questions buried in the noise

Community team should triage Critical by category before mass-actioning. Amber (25) is the clean working list.

---

## Post URL Format

⚠️ Needs validation with community team. Best-guess for Khoros/Lithium platform:
```
https://community.jamf.com/t5/[board-slug]/[title-slug]/td-p/[ID_TOPIC_PUBLIC]
```
Until confirmed, `ID_TOPIC_PUBLIC` is included in output so posts can be looked up manually.

---

## Edge Cases

| Scenario | Handling |
|---|---|
| `TRY_TO_TIMESTAMP` returns NULL | Exclude row |
| Question < 7 days old | Excluded by filter |
| Question has replies but no formal answer event | Surfaces in queue — reply ≠ answer |
| Spam flag false positive | Flag is ⚠️ review prompt, not auto-remove |
| Red or Critical requested | Re-run query filtered to that band, oldest first |
| Empty result | "No unanswered questions older than 7 days. 🎉" |

---

## Hard Rails

- Does NOT delete, hide, or mark posts as spam.
- Does NOT post replies or mark questions answered.
- Does NOT auto-action flagged posts — ⚠️ is for human review only.
- Does NOT surface Red/Critical unprompted — counts only unless asked.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| v1.0 | 2026-05-11 | Spec written — build blocked on schema validation. |
| v1.1 | 2026-05-12 | Schema confirmed. Production query written. |
| v1.2 | 2026-05-12 | Schema corrected to GAINSIGHT_CC. Output → chat table. Reply count from CONTENT_TOPIC_REPLIED. |
| v1.3 | 2026-05-12 | Output redesigned: Amber full list (newest first) + bucket counts for Red/Critical. Spam flag added (heuristic). Volume note added. |

---

*2026.06.13 — Ported from the Pulse archive package; scrubbed, CalVer.*
