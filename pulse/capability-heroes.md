# Pulse — Capability: Heroes Tracker & Pipeline
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/pulse/capability-heroes.md
#
# Two capabilities in one file — routed by trigger.
# Heroes tracker: current Heroes health and readiness scores
# Heroes pipeline: candidates to recruit as new Heroes

---

## Routing

| Trigger | Variant |
|---------|---------|
| "Heroes tracker" / "Heroes health" / "how are our Heroes doing" / "super users" | `super_users` |
| "Heroes candidates" / "Heroes pipeline" / "who should we recruit" / "Heroes pipeline" | `heroes_pipeline` |

---

## Readiness Score formula (canonical — also in protocol.md)

```
Reply Score     = LEAST(replies_365d / 100.0, 1.0) * 40
Solution Score  = LEAST(solution_milestone / 50.0, 1.0) * 30
Recency Score   = (replies_90d / NULLIF(replies_365d, 0)) * 30
Readiness Score = LEAST(Reply Score + Solution Score + Recency Score, 100.0)
```

Both capabilities use the same formula. Data coverage caveat: CC data only from August 2024. Always surface.

---

---

# HEROES TRACKER (`super_users`)

Current Heroes — health and readiness across the active Heroes roster.

Load `references/super-users.md` for full query spec, badge milestone CTEs, and HTML template.

**Inputs from badge milestone CTEs:**
- `replies_365d` — replies authored in last 365 days
- `solution_milestone` — solutions authored (from badge data)
- `replies_90d` — replies authored in last 90 days (recency signal)

**Join notes:**
- `USER_BADGE.ID_USER = COMMUNITY_USER.CAT_USER_NUMBER`
- `BADGE` join: `b.ID_BADGE = ub.ID_BADGE` (not `b.ID = ub.ID_BADGE_ID`)
- `USER_BADGE.TM_RECEIVED` is TIMESTAMP — do not wrap in `TRY_TO_TIMESTAMP()`
- CC badge dates cluster June 26, 2025 (migration artifacts) — recency scoring currently disabled

**Output:** HTML artifact — `templates/super-users-template.html`.

---

---

# HEROES PIPELINE (`heroes_pipeline`)

Candidates to recruit as Heroes — users not yet in the program who meet readiness threshold.

Load `references/heroes-pipeline.md` for full query spec and pipeline threshold.

**Threshold:** Readiness Score ≥ 35 (minimum bar for outreach).

**Key filters:**
- Exclude current Heroes
- Exclude internal Jamf employees
- Apply `CAT_CSM_NAME != 'Jamf Digital Team'` exclusion
- Min `replies_365d` > 0 (must have recent activity)

**Output:** Chat table.

```
| Name | Company | Readiness Score | Band | Replies (365d) | Solutions |
|------|---------|----------------|------|----------------|-----------|
```

Sort by Readiness Score descending.

---

*2026.06.13 — CalVer header adopted. Readiness Score formula, tracker/pipeline logic, joins, and outputs unchanged.*
*2026.06 (v1.0) — Extracted from Pulse v1.1. Full query specs in references/super-users.md and references/heroes-pipeline.md. Readiness Score formula canonical in protocol.md.*
