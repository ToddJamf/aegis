# Aegis
Hosted logic layer for Jamf CS Ops Claude skills.
**Repo:** https://github.com/ToddJamf/aegis (public)
**Owner:** Todd Massey, CS Ops

Skills contain a thin bootstrap SKILL.md that fetches live instructions from this repo at runtime. Update a file here, push to main — all users pick it up automatically, no reinstall.

---

## ⛔ Repo guardrails — read before committing

This repo is **public**. A hard rule governs what may live here:

**Allowed (short-term):** Jamf internal IDs, Gainsight field/schema structure, internal Slack channel IDs. Accepted trade-off — data hosting moves elsewhere and this repo goes private later.

**NEVER commit — no exceptions:** customer information, PII (including employee names/emails), or compliance data.

Before committing any file — especially bundled references ported from legacy sentinel/Sentinel-era packages — scan for and scrub customer names, ARR figures, contact PII, employee names/emails, community usernames, and compliance data. Genericize examples to fictional names (Acme, etc.).

**The gate is reading, not grep.** A known-name grep is necessary but NOT sufficient — it only catches names already on the list, and legacy archives carry names you've never seen (real customers, employees, community handles all slipped a grep-only pass during the rebuild). Anything ported from a legacy `.skill` archive must be **read and scrubbed at the source before it enters the repo**, then verified by a line-by-line read — not a blocklist grep. Use example.com for any sample email; never a real domain. Run the grep too (`@`, `$`, known names) as a backstop, not the primary check.

---

## Skills served

| Skill | Purpose | Bootstrap |
|-------|---------|-----------|
| **Oracle** | Account meeting prep: pre-call brief, success-plan draft | `oracle/SKILL.md` |
| **Mentor** | People prep: 1:1 prep (down to a report, up to your manager) | `mentor/SKILL.md` |
| **Radar** | Operational CS: renewal, expansion, plate, gaps, compliance, heat-map, portfolio | `radar/SKILL.md` |
| **Pulse** | Community intelligence: dashboard, dormant, unanswered, Heroes | `pulse/SKILL.md` |

Quick single-fact lookups ("when does Acme renew?", "who's the CSM on Acme?") are served by the Gainsight MCP connector directly — no dedicated skill. Glossary ("what is a CTA?") is owned by Oracle.

---

## Shared layer (the lean architecture)

Cross-cutting logic lives once in `shared/` and is fetched by every skill — fix once, all skills pick it up:

```
shared/
  identity.md          ← who's asking: 6-tier CS model, org walk, verify_downward, detect_direction
  source-routing.md    ← which tool answers which data read (incl. Bluhm B7 Staircase routing)
  error-handling.md    ← Recover / Degrade / Escalate; §0 access-escalation foundation
  output-discipline.md ← footer policy, CalVer, voice, anti-AI language (one SSOT)
  redirect.md          ← cross-skill dispatch: who-owns-what map + redirect pattern
```

A skill only knows "is this mine?" — everything cross-cutting (data routing, errors, footer, handoff) is delegated to the shared layer, not reimplemented locally.

---

## Structure

```
aegis/
  shared/   identity · source-routing · error-handling · output-discipline · redirect
  oracle/   SKILL · protocol · capability-brief · capability-plan · references/ (field-registry, write-gateway, arr-policy, run-as, menus, glossary, terminology-cs/public, synonym-map, overview, test-mode, connector-setup, push, fallback)
  mentor/   SKILL · protocol · capability-1on1 · references/ (field-registry, fallback)
  radar/    SKILL · protocol · capability-renewal/expansion/plate/gaps/heat-map/portfolio · references/ (identity, scoring, artifact, field-registry, arr-policy, connector-setup, fallback)
  pulse/    SKILL · protocol · capability-dashboard/dormant/unanswered/heroes
```

(The `sentinel/` monolith was removed 2026.06.13 — fully superseded by the carved skills above.)

---

## Skill scope boundaries

The authoritative who-does-what map lives in `shared/redirect.md`. Summary:

| Request | Skill |
|---------|-------|
| "Brief me on Acme" / "I have a call with Acme" / "Draft a success plan for Acme" | Oracle |
| "1:1 with [name]" / "prep for my 1:1 with [manager]" | **Mentor** |
| "Show my radar" / "what's renewing" / "what's on my plate" / "team compliance" / "heat map" | Radar |
| "When does Acme renew?" / "who is the CSM on Acme?" | Gainsight connector directly (or Oracle for a full brief) |
| "What is a CTA?" / glossary | Oracle |
| "Community dashboard" / "dormant accounts" / "Heroes tracker" | Pulse |
| ARR trends, license analytics, Snowflake data | ask-snowflake-analyst |

---

## Identity model

**CS org identity — `shared/identity.md`** (Oracle, Mentor, Radar leader capabilities):
6-tier model — CSM · FLL · Segment Leader · SR Leadership · Department Head · Not CS. No-record/no-access → error-handling §0 escalate (no dead-end).

**Radar CSM/AE identity — `radar/references/identity.md`:** persona detection via book counts (Csm vs `Primary_Territory_Owner__gc`). Distinct from CS org identity. *(Pending: this file needs to be brought into the repo — see Outstanding.)*

---

## Update workflow

1. Edit files in `~/Documents/GitHub/aegis/`
2. Open GitHub Desktop → Commit → Push
3. Done — all users pick up on next activation

---

## Versioning — CalVer

One stack date (`Aegis stack YYYY.MM.DD`) per ship, per-file changelog footers, git tag per ship. The legacy `Aegis vN` scheme is retired — see `shared/output-discipline.md` §2.

---

## Outstanding

- [ ] `shared/identity.md` — swap hardcoded book-size query for `get_portfolio_filters` (gated on a real CSM-seat probe, Decision 8)
- [ ] Repackage Oracle + Mentor + Radar + Pulse `.skill` files; UAT (FLL + Segment Leader cold-run)
- [ ] Uninstall the `gainsight-sentinel` monolith AND the standalone Ask Gainsight skill (folded into Oracle) from the installed set
- [ ] (Deferred) Radar `update` capability — never built; write-adjacent, revisit post-Phase-2
- [ ] Centralized agentic deployment (future scope)

---

*Aegis stack 2026.06.13 | Oracle · Mentor · Radar · Pulse*
