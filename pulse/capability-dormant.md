# Pulse — Capability: Dormant Accounts
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-dormant.md
#
# Triggers: "dormant accounts", "accounts gone quiet", "re-engagement list", "who's gone dark"

---

## Definitions

- **Dormant:** Was active (had ≥1 engagement event), gone quiet ≥90 days. Not the same as Dark.
- **Dark:** Never active. Different motion — not covered by this capability.
- **Engagement event:** LIKE or REPLY. Views are NOT engagement events.
- **Internal exclusion:** Always apply `CAT_CSM_NAME != 'Jamf Digital Team'`.

Full locked definitions in `references/kpi-definitions.md`.

---

## Query logic

Load `references/dormant.md` for the full query spec, ARR join, edge cases, and output template.

Key elements:
- Source: `STG__GAINSIGHT__COMMUNITY_ENGAGEMENT_OVERVIEW` + `STG__GAINSIGHT__COMPANY_ALL_COLUMNS`
- Join on `CAT_NAME = CAT_COMPANY_NAME` for ARR
- Dormant filter: last activity 90–179 days ago (⚠️ orange) and ≥180 days ago (🚨 red)
- ARR tier from `COMPANY_ALL_COLUMNS.ID_ARR_BAND`

**ARR Band GSID Map:**

| Label | GSID |
|---|---|
| $10k or less | `1I00ONSM92NZS4XF7Q5CLYL0M3D6SU9AMDR9` |
| $10k – $50k | `1I00ONSM92NZS4XF7QIRWZNOF3IFFF6YNTTV` |
| $50k – $100k | `1I00ONSM92NZS4XF7Q5DS0FHZUXUDW2KCYB7` |
| $100k – $250k | `1I00ONSM92NZS4XF7QBPF2L1XWTVY15Y2H2Y` |
| $250k or more | `1I00ONSM92NZS4XF7Q8LN2ZJZM9AHDVBX6RR` |
| Unknown | Any other value or NULL |

Always include Unknown footnote: *"Unknown = no ARR data on file for this account in Gainsight."*

---

## Output

HTML artifact — `templates/dormant-template.html`.

Columns: Company · CSM · ARR Tier · Last Activity · Days Dormant · Dormancy Band (⚠️/🚨)

---

*2026.06.13 — CalVer header adopted. Query spec, ARR band map, and output unchanged.*
*2026.06 (v1.0) — Extracted from Pulse v1.1. Full query spec in references/dormant.md.*
