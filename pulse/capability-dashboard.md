# Pulse — Capability: Community Dashboard
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-dashboard.md
#
# Triggers: "community dashboard", "KPI report", "how is Jamf Nation doing",
#           "community health", "community KPIs", "community snapshot"
# Ops variant: "ops dashboard", "weekly ops"

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

Trigger: "ops dashboard" / "weekly ops"

Output: HTML — reuse the standard dashboard render (`references/dashboard.md`). A dedicated ops-dashboard template is a deferred enhancement (not yet authored).

---

*2026.06.13 — PII scrub: employee names removed from the ops-dashboard triggers (now "ops dashboard" / "weekly ops"). CalVer header adopted. Render specs unchanged.*
*2026.06 (v1.0) — Extracted from Pulse v1.1 SKILL.md. Full render spec in references/dashboard.md and references/pdf-template.html.*
