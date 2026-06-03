# Pulse — Capability: Community Dashboard
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-dashboard.md
#
# Triggers: "community dashboard", "KPI report", "how is Jamf Nation doing",
#           "community health", "community KPIs", "community snapshot"
# Ops variant: "ops dashboard", "weekly ops", "Mitchell dashboard", "Jeni dashboard"

---

## Dashboard (`dashboard`)

Full community health report. Load `references/dashboard.md` for the complete render spec and SQL.

Output: PDF via `references/pdf-template.html`.

**Core KPIs:**
- Active accounts (30-day window)
- Engagement rate (engaged / registered)
- Total engagement events (likes + replies — NOT views)
- New registrations
- Unanswered question count
- Heroes tracker summary

**Partial month:** Flag all current-month labels with ★. Never extrapolate.

---

## Ops Dashboard (`pulse_ops`)

Trigger: "ops dashboard" / "weekly ops" / "Mitchell dashboard" / "Jeni dashboard"

Output: HTML via `references/ops-dashboard-template.html`. Load that file for full spec.

---

*Pulse capability-dashboard.md v1.0 (2026-06-03) — Extracted from Pulse v1.1 SKILL.md. Full render spec in references/dashboard.md and references/pdf-template.html.*
