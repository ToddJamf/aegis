# Pulse — Protocol (Router)
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/protocol.md
#
# Pulse is Jamf's community intelligence skill. Surfaces Jamf Nation health,
# engagement trends, dormant accounts, unanswered posts, Heroes tracker, and Heroes pipeline
# from Snowflake. All users — no Gainsight identity gate required.

---

## Does NOT do

- **No Gainsight CS data** — account health, CTAs, renewal → Oracle or Ask Gainsight
- **No ARR trend analytics** → ask-snowflake-analyst
- **No external posts or sends** without explicit user confirmation
- **No writes** to Gainsight or any system

---

## Output discipline

- Tool loading is silent. Never narrate intermediate steps.
- First user-facing line is the answer.
- Partial month data: always flag with ★ on chart labels. Never extrapolate partial-month data to full-month projections.
- **AI disclaimer — always present:** Short variant in headers. Full variant in footers:
  *"AI-generated from Snowflake/Gainsight. Verify before acting."*

---

## Data sources

| Schema | Table | Notes |
|---|---|---|
| `BI_CURATED.STG__GAINSIGHT` | `STG__GAINSIGHT__COMMUNITY_ENGAGEMENT_OVERVIEW` | Primary account-level table |
| `BI_CURATED.STG__GAINSIGHT` | `STG__GAINSIGHT__COMMUNITY_USER` | User registration, roles (CAT_ROLES), badges |
| `BI_CURATED.STG__GAINSIGHT` | `STG__GAINSIGHT__COMPANY_ALL_COLUMNS` | Company-level ARR |
| `BI_CURATED.GAINSIGHT_CC` | `CONTENT_REPLY_LIKED` | Reply likes. Timestamp: `TM_OCCURED` (one R — schema typo) |
| `BI_CURATED.GAINSIGHT_CC` | `CONTENT_TOPIC_LIKED` | Topic likes. Timestamp: `TM_OCCURED` (one R) |
| `BI_CURATED.GAINSIGHT_CC` | `CONTENT_TOPIC_REPLIED` | Topic replies. Timestamp: `TM_OCCURRED` (two Rs) |
| `BI_CURATED.GAINSIGHT_CC` | `CONTENT_TOPIC_CREATED` | Topics created. `TM_OCCURRED` stored as TEXT — cast to TIMESTAMP |
| `BI_CURATED.GAINSIGHT_CC` | `CONTENT_QUESTION_ANSWERED` | Answer events. Join key: ID_TOPIC_PUBLIC |
| `BI_CURATED.GAINSIGHT_CC` | `USER_BADGE` | Badge awards per user |
| `BI_CURATED.GAINSIGHT_CC` | `BADGE` | Badge metadata |

**CC event tables start August 2024.** Pre-Aug 2024 history requires `STG__GAINSIGHT` tables.

**Internal exclusion — always apply:**
```sql
WHERE CAT_CSM_NAME != 'Jamf Digital Team'
```

---

## Reference loading (progressive disclosure)

Load on demand.

| File | Load when | Tier |
|------|-----------|------|
| `references/kpi-definitions.md` | Any query needing metric definitions or exclusion rules | 🔴 Blocking |
| `https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-dashboard.md` | Dashboard / KPI report requested | 🔴 Blocking |
| `https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-dormant.md` | Dormant accounts requested | 🔴 Blocking |
| `https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-unanswered.md` | Unanswered posts requested | 🔴 Blocking |
| `https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-heroes.md` | Heroes tracker / pipeline requested | 🔴 Blocking |
| `references/dashboard.md` | Dashboard render spec | 🔴 Blocking |
| `references/dormant.md` | Dormant query spec (supplementary) | 🟡 Degraded |
| `references/unanswered.md` | Unanswered posts query spec (supplementary) | 🟡 Degraded |
| `references/super-users.md` | Heroes tracker spec (supplementary) | 🟡 Degraded |
| `references/heroes-pipeline.md` | Heroes pipeline spec (supplementary) | 🟡 Degraded |

**Fallback:**
- 🔴 Blocking — load fails → hard stop. *"Pulse is missing a required file. Contact CS Ops."*
- 🟡 Degraded — load fails → proceed with warning.

---

## Step 0 — MCP inventory

- **Snowflake** (`sql_exec_tool` callable) — required for all data capabilities. If absent: *"Pulse needs the Snowflake connector. Go to Cowork → Settings → Connectors → Snowflake → Always allow."* Glossary lookups still work.

---

## Step 2 — Route the query

| Trigger | Capability |
|---------|-----------|
| "community dashboard" / "KPI report" / "how is Jamf Nation doing" / "community health" / "community KPIs" | `dashboard` |
| "ops dashboard" / "weekly ops" / "Mitchell dashboard" / "Jeni dashboard" | `pulse_ops` |
| "dormant accounts" / "accounts gone quiet" / "re-engagement list" / "who's gone dark" | `dormant` |
| "unanswered questions" / "unanswered posts" / "what needs a reply" / "posts with no answer" | `unanswered` |
| "Heroes tracker" / "Heroes health" / "how are our Heroes doing" / "super users" | `super_users` |
| "Heroes candidates" / "Heroes pipeline" / "who should we recruit" / "Heroes pipeline" | `heroes_pipeline` |
| "what is [community term]" / "explain [Jamf Nation concept]" | Inline glossary — load `references/kpi-definitions.md` |

---

## Output medium

| Result shape | Format |
|--------------|--------|
| 1–2 metrics | Chat |
| Full dashboard | HTML render |
| Leadership report | PDF via `references/pdf-template.html` |
| Ops report (Mitchell/Jeni) | HTML via `references/ops-dashboard-template.html` |
| Dormant account list | HTML artifact — `templates/dormant-template.html` |
| Unanswered posts queue | Chat table — full titles, IDs, links |
| Heroes tracker | HTML artifact — `templates/super-users-template.html` |
| Heroes pipeline | Chat table |

---

## Heroes Readiness Score formula (canonical — shared by both heroes capabilities)

```
Reply Score     = LEAST(replies_365d / 100.0, 1.0) * 40
Solution Score  = LEAST(solution_milestone / 50.0, 1.0) * 30
Recency Score   = (replies_90d / NULLIF(replies_365d, 0)) * 30
Readiness Score = LEAST(Reply Score + Solution Score + Recency Score, 100.0)
```

**Score bands:**

| Score | Label | Color |
|---|---|---|
| 70–100 | Active Hero | Freshmaker `#50DEAF` |
| 40–69 | Fading | Amber `#E08A00` |
| 10–39 | At Risk | Red `#C41E3A` |
| 0–9 | Inactive | Gray `#9CA3AF` |

**Caveat:** Scores reflect CC activity since August 2024. Tenured contributors may score lower due to earlier history not captured in CC tables. Always surface this caveat.

---

## Known data quality issues (summary — full spec in `references/kpi-definitions.md`)

| Issue | Fix |
|---|---|
| `TM_OCCURED` one R on REPLY_LIKED and TOPIC_LIKED | Use `TM_OCCURED` (one R) |
| `TM_OCCURRED` two Rs on TOPIC_REPLIED | Use `TM_OCCURRED` (two Rs) |
| `TM_OCCURRED` TEXT on TOPIC_CREATED and QUESTION_ANSWERED | `TRY_TO_TIMESTAMP(TM_OCCURRED)` |
| CC tables start Aug 2024 | Union with STG__GAINSIGHT for pre-Aug 2024 trends |
| CC schema has no STG__ prefix | `BI_CURATED.GAINSIGHT_CC` — not `STG__GAINSIGHT_CC` |
| N_REPLIES not in CONTENT_TOPIC_CREATED | Count rows in CONTENT_TOPIC_REPLIED instead |
| USER_BADGE join key is ID_USER | `ID_USER = COMMUNITY_USER.CAT_USER_NUMBER` |
| BADGE join columns are ID_BADGE | Both sides: `b.ID_BADGE = ub.ID_BADGE` |
| CC badge dates are migration artifacts | All cluster June 26, 2025 — recency scoring disabled |

---

## Visual brand

```
Celtic Blue  #056AE6  — primary accent, bars, links
Haiti        #100F2F  — headers, body text
Pearl        #F9F7F1  — card backgrounds
Deep Sea     #00348A  — secondary data color
Blue Violet  #5050BC  — tertiary data
Freshmaker   #50DEAF  — positive signals (Active Hero)
Amber        #E08A00  — warnings, partial data (Fading)
Red          #C41E3A  — errors, critical (At Risk)
Gray         #9CA3AF  — inactive
```

Typography: Inter (primary) · IBM Plex Mono (data labels, metrics, code)

---

## Blocked capabilities

| Capability | Blocker |
|---|---|
| `advocates` | NPS field name/table in Gainsight not confirmed |
| `pulse_csm` full build | Gainsight identity resolution not tested |
| `trending` | Phase 2 |

---

*Pulse v1.0 Aegis (2026-06-03) — Ported from Pulse v1.1. Heroes Readiness Score formula, data quality issues, and ARR Band GSID map moved here from SKILL.md for centralised maintenance. Capability logic extracted to capability-*.md files.*
