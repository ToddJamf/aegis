---
name: pulse
description: "Pulse is Jamf's community intelligence skill. Use it whenever anyone asks about Jamf Nation community health, engagement, active accounts, dormant accounts, unanswered posts, super-users, community KPIs, community trends, or wants a community report or dashboard. Also trigger when a CSM asks about their book's community activity, or when CS leadership asks about community metrics. Triggers: 'community dashboard', 'community health', 'community KPIs', 'how is Jamf Nation doing', 'community engagement', 'community report', 'dormant accounts', 'unanswered posts', 'Heroes tracker', 'Heroes pipeline', 'Heroes candidates', 'who should we recruit for Heroes', 'top contributors', 'pulse weekly', 'pulse csm', 'replies per topic', 'community snapshot'. ALWAYS use this skill for any Jamf Nation community data request — do not query Snowflake or interpret community data without reading this skill first."
---

# Pulse — Community Intelligence

*Jamf Nation health, engagement, dormant accounts, unanswered posts, and Heroes intelligence from Snowflake.*

---

## On activation

Fetch and follow:

```
https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/protocol.md
```

That file is the authoritative source of truth for Pulse's capabilities, routing, data sources, output discipline, known data quality issues, and the Heroes Readiness Score formula. Follow its instructions exactly.

**If the fetch fails:** Load `references/kpi-definitions.md` for data source and metric definitions, then answer with best available signal. Warn: *"⚠ Pulse couldn't load its full instructions — output may be incomplete."*

---

## Bundled reference files

Load on demand per the reference loading table in `protocol.md`:

- `references/kpi-definitions.md` — locked metric definitions, exclusion rules, data quality issues
- `references/dashboard.md` — KPI report render spec
- `references/pdf-template.html` — KPI report HTML → PDF template
- *(footer/disclaimer policy lives in `shared/output-discipline.md` — full footer + header short-form. Don't author Pulse-local disclaimer strings.)*
- `references/dormant.md` — dormant account query spec (v2.1)
- `references/unanswered.md` — unanswered posts query spec (v1.3)
- `references/super-users.md` — Heroes tracker spec (v1.3)
- `references/heroes-pipeline.md` — Heroes pipeline spec (v1.3)
- `templates/dormant-template.html` — dormant HTML render template
- `templates/unanswered-template.html` — unanswered HTML render template
- `templates/super-users-template.html` — Heroes tracker HTML template
- `templates/heroes-pipeline-formatter.md` — Heroes pipeline chat output formatter

---

*Pulse 2026.06.13 — Community intelligence skill on the Aegis shared layer. Capabilities: dashboard · ops dashboard · dormant · unanswered · Heroes tracker · Heroes pipeline. Footer/voice governed by shared/output-discipline.md; cross-skill handoff by shared/redirect.md. Community KPIs and Snowflake sources unchanged.*
*2026.06 (v2.0) — Ported to Aegis hosted logic framework. Logic extracted to aegis/pulse/. Bundled references unchanged from v1.1.*
