# Aegis
Hosted logic layer for Jamf CS Ops Claude skills.
**Repo:** https://github.com/ToddJamf/aegis (public)
**Owner:** Todd Massey, CS Ops

Skills contain a thin bootstrap SKILL.md that fetches live instructions from this repo at runtime. Update a file here, push to main — all users pick it up automatically, no reinstall.

---

## Skills served

| Skill | Purpose | Bootstrap |
|-------|---------|-----------|
| **Oracle** | Meeting prep: brief, 1:1, success plan | `oracle/SKILL.md` |
| **Radar** | Operational CS: renewal, expansion, plate, gaps, compliance, heat-map, portfolio, update | `radar/SKILL.md` |
| **Ask Gainsight** | Quick factual lookup, all users | `ask-gainsight/SKILL.md` |
| **Pulse** | Community intelligence: dashboard, dormant, unanswered, Heroes | `pulse/SKILL.md` |

---

## Structure

```
aegis/
  shared/
    identity.md              ← 6-tier CS identity (Oracle, Ask Gainsight, Pulse)
    error-handling.md        ← shared error patterns
  oracle/
    SKILL.md                 ← thin bootstrap
    protocol.md              ← router + orchestration (meeting prep)
    capability-brief.md      ← pre-call brief
    capability-1on1.md       ← 1:1 prep (DOWN/CSM, DOWN/FLL, UP)
    capability-plan.md       ← success plan draft
  radar/
    SKILL.md                 ← thin bootstrap
    protocol.md              ← router (renewal + expansion + operational)
    capability-renewal.md    ← renewal triage (R1–R5 scoring)
    capability-expansion.md  ← expansion pipeline (E1–E4 scoring)
    capability-plate.md      ← my-plate (CSM) + team plate (FLL)
    capability-gaps.md       ← my-gaps (CSM) + compliance (FLL)
    capability-heat-map.md   ← team risk, escalations, wins (FLL+)
    capability-portfolio.md  ← portfolio / segment story (FLL+)
  ask-gainsight/
    SKILL.md                 ← thin bootstrap
    protocol.md              ← quick lookup, glossary, all users
  pulse/
    SKILL.md                 ← thin bootstrap
    protocol.md              ← router + data quality + Heroes score formula
    capability-dashboard.md  ← community KPI report + ops dashboard
    capability-dormant.md    ← dormant account list
    capability-unanswered.md ← unanswered posts queue
    capability-heroes.md     ← Heroes tracker + Heroes pipeline
  sentinel/                  ← archived (superseded by Oracle + Radar + Ask Gainsight)
```

---

## Skill scope boundaries

| Request | Skill |
|---------|-------|
| "Brief me on Acme" / "I have a call with Acme" | Oracle |
| "1:1 with [name]" / "prep for my 1:1 with [manager]" | Oracle |
| "Draft a success plan for Acme" | Oracle |
| "Show my radar" / "what's renewing" / "expansion pipeline" | Radar |
| "What's on my plate" / "top accounts this week" | Radar |
| "Activity gap" / "team compliance" / "who needs coaching" | Radar |
| "What should I escalate" / "team heat map" / "segment story" | Radar |
| "When does Acme renew?" / "who is the CSM on Acme?" | Ask Gainsight |
| "What is a CTA?" / "what is a success plan?" | Ask Gainsight |
| "Community dashboard" / "dormant accounts" / "Heroes tracker" | Pulse |
| ARR trends, license analytics, Snowflake data | ask-snowflake-analyst |

---

## Identity model

**CS org identity — shared/identity.md (Oracle, Radar leader capabilities, Ask Gainsight):**
Full 6-tier model: CSM · FLL · Segment Leader · SR Leadership · Department Head · Not CS

**Radar CSM/AE identity — radar/references/identity.md:**
Persona detection via book counts (Csm vs Primary_Territory_Owner__gc field). Distinct from CS org identity.

---

## Update workflow

1. Edit files in `~/Documents/GitHub/aegis/`
2. Open GitHub Desktop → Commit → Push
3. Done — all users pick up on next activation

---

## Bootstrap pattern

Each skill's SKILL.md:
1. Carries `name` + `description` for triggering
2. Fetches `protocol.md` from Aegis on activation
3. Lists bundled `references/` files available locally

**Aegis-hosted:** routing logic, capability workflows, query patterns (change with business logic)
**Bundled:** field definitions, terminology, templates, GSID maps, HTML specs (stable reference data)

---

## Outstanding

- [ ] Rename Gainsight Sentinel → Oracle (new .skill package)
- [ ] Rebuild Radar .skill package with expanded SKILL.md
- [ ] Build Ask Gainsight .skill package
- [ ] Rebuild Pulse .skill package with updated SKILL.md
- [ ] Centralized agentic deployment (future scope)

---

*Aegis v2.0 — June 2026 | Oracle · Radar · Ask Gainsight · Pulse*
