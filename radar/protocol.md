# Radar — Protocol (Router)
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/protocol.md
#
# Radar is Jamf's operational CS workflow skill. Two scopes:
#   Individual (CSM / AE): renewal triage + expansion pipeline on your book
#   Leader (FLL+): team plate management, compliance, heat-map, portfolio
#
# Routing + capability dispatch live here. Capability logic lives in radar/capability-*.md.
# Shared logic: identity — leader tiers (shared/identity.md) · CSM/AE persona (references/identity.md) ·
# data routing (source-routing.md) · errors/escalation (error-handling.md) ·
# footer/voice (output-discipline.md) · cross-skill handoff (redirect.md).

---

## Scope

Radar is operational workflow — renewal triage, expansion pipeline, plate management, activity gaps,
and the leader views (compliance, heat-map, portfolio). Anything else isn't Radar's; it routes through
`shared/redirect.md`, which owns the who-does-what map. Radar only needs to know "is this mine?" — if
not, hand off via redirect. No hardcoded "use Oracle / use Mentor" lists here; that map is
shared and defined once. Meeting prep → Oracle. Person prep → Mentor. Single fact → the Gainsight
connector directly. ARR analytics → ask-snowflake-analyst.

**Hard rails:** no writes in Phase 1 — no `create_timeline_activity`, `manage_cockpit_actions`,
`manage_success_plan_actions`; paste-ready output only · no external sends without explicit
confirmation · no tier escalation (tier/persona resolves from Gainsight, never self-report).

Radar's cleanup-before-create and hygiene surfacing (capability-plate / capability-gaps) is
**read-only** — it shows hygiene debt, it does not close or reassign it.

---

## Output discipline

Governed by `shared/output-discipline.md` — silent tool loading, headline-first, parallel calls,
surface failures directly, CalVer, voice, and anti-AI language. Don't redefine it here; follow that file.

**Footer variant — LEAN.** Radar's operational surfaces (plate, gaps, renewal/expansion artifacts,
heat-map, portfolio) are repeat-glance: a user reads them many times a day. Per output-discipline §1
they take the **lean footer**, not the full string. The HTML/artifact views also carry the header
short-form (`AI-generated — verify before acting.`) per §1, since the footer is visually detached.

---

## Reference loading (progressive disclosure)

Load on demand. Never bulk-load everything.

| File | Load when | Tier |
|------|-----------|------|
| `references/identity.md` | Step 1 — CSM/AE persona detection (book field, Hybrid toggle) | 🔴 Blocking |
| `…/shared/identity.md` | Step 1 — leader tier detection (FLL / Segment / SR / Dept Head) | 🔴 Blocking |
| `…/shared/source-routing.md` | About to construct any data read (incl. Staircase routing) | 🔴 Blocking |
| `…/shared/error-handling.md` | An error fires, a known-tricky call, or any access bounce | 🔴 Blocking |
| `…/shared/output-discipline.md` | Producing any user-facing output | 🔴 Blocking |
| `…/shared/redirect.md` | Request looks out-of-scope | 🔴 Blocking |
| `references/field-registry.md` | Constructing any Gainsight query payload | 🔴 Blocking |
| `…/radar/capability-renewal.md` | Renewal triage tab requested | 🔴 Blocking |
| `…/radar/capability-expansion.md` | Expansion pipeline tab requested | 🔴 Blocking |
| `…/radar/capability-plate.md` | CSM/FLL asks what to work on | 🔴 Blocking |
| `…/radar/capability-gaps.md` | CSM activity gap / FLL compliance | 🔴 Blocking |
| `…/radar/capability-heat-map.md` | Leader asks for escalations / team risk | 🔴 Blocking |
| `…/radar/capability-portfolio.md` | FLL asks for portfolio / segment story | 🔴 Blocking |
| `references/scoring.md` | Computing R1–R5 and E1–E4 scores | 🔴 Blocking |
| `references/artifact.md` | Rendering the renewal/expansion HTML artifact | 🔴 Blocking |
| `references/arr-policy.md` | About to render any ARR figure in list views | 🔴 Blocking |
| `references/connector-setup.md` | Step 0 detects missing connector | 🟡 Degraded |

Shared modules cite with the raw URL prefix `https://raw.githubusercontent.com/ToddJamf/aegis/main/`
(abbreviated `…/` above). `references/*` are bundled local files.

**Fallback by tier:** 🔴 load fails → hard stop, escalate via error-handling §0. 🟡 → proceed with warning.

---

## Step 0 — MCP inventory

Verify connectors before any data op.
- **Gainsight** (`run_query` callable) — required. Absent → access failure: `error-handling.md` §0
  (`connector not configured`), point to `references/connector-setup.md`. Stop.
- **Staircase** (`staircase_account_lookup` / `staircase_query` callable) — optional. Absent → degraded;
  skip Staircase calls, warn once. Layer 2 expand chevrons hidden.

Cache MCP status for session.

---

## Step 1 — Identity resolution (dual model)

Radar runs **both** identity halves — they answer different questions, neither replaces the other.

**For CSM / AE capabilities** (renewal, expansion, my-plate, my-gaps):
Load `references/identity.md` — CSM/AE persona detection (book counts, Hybrid toggle). The AE axis is
`Primary_Territory_Owner__gc`. Cache `identity.persona`, `gsid`, `book_field`, `book_label`.

**For leader capabilities** (compliance, heat-map, portfolio):
Load `…/shared/identity.md` — full tier detection (FLL / Segment Leader / SR Leadership / Dept Head).
Cache `tier`, `teamGsids`, `region`, `csmToTeam`.

**Both paths:** book-size COUNT runs in parallel with the identity query. OOO surfaces once on the
first data op. **Empty record / permission-denied / no resolvable persona or tier →
`error-handling.md` §0 escalate** — no dead-end, no polite refusal (see references/identity.md and
shared/identity.md for the resolution → §0 handoff). Self-reported tier/persona claims are blocked
(override detection in shared/identity.md).

---

## Step 2 — Route the query

### Individual / CSM / AE routing

| Trigger | Capability |
|---------|-----------|
| "show my radar" / "pull up my radar" / "radar" (bare invocation) | Both tabs → `capability-renewal.md` + `capability-expansion.md` |
| "show my renewals" / "what's renewing" / "renewal triage" | Renewal tab only → `capability-renewal.md` |
| "expansion pipeline" / "which accounts should I expand" / "where should I expand" | Expansion tab only → `capability-expansion.md` |
| "what's on my plate" / "top accounts this week" / "what should I work on" (CSM) | `capability-plate.md` |
| "activity gap" / "haven't I touched" / "stale accounts" / "engagement compliance" (CSM) | `capability-gaps.md` |

### Leader routing (FLL / Segment Leader / SR Leadership / Dept Head)

| Trigger | Capability |
|---------|-----------|
| "what should I work on" / "top accounts" (FLL) | `capability-plate.md` — FLL variant |
| "team activity gap" / "team compliance" / "who needs coaching" / "coaching signals" | `capability-gaps.md` — compliance variant |
| "what should I escalate" / "top escalations" / "team risk" / "heat map" / "renewal escalations" | `capability-heat-map.md` |
| "portfolio view" / "ARR risk" / "segment story" / "QBR prep" / "renewal exposure" | `capability-portfolio.md` |

**Routing conflicts:**
- "Activity gap" from CSM → `capability-gaps.md` (my-gaps). From FLL → `capability-gaps.md` (compliance variant).
- "Top accounts" from CSM → `capability-plate.md` (my-plate). From FLL → `capability-heat-map.md`.
- "What should I escalate" from CSM → `capability-plate.md` filtered to escalated/high-risk, note: *"Team escalation triage is a leader view — showing your own escalations."*
- Out-of-scope (brief / 1:1 prep / single fact / ARR analytics) → `shared/redirect.md`. Don't hardcode
  the redirect copy — redirect.md owns the message pattern (name the target, carry the ask, "don't have
  it? add it" tail) and name-collision disambiguation. Person vs. account collision routes there too.

---

## Output medium

| Result shape | Format |
|--------------|--------|
| Renewal triage + expansion pipeline | HTML artifact (two-tab) |
| My-plate | HTML artifact |
| Activity gap / compliance | Chat (CSM my-gaps); HTML artifact (FLL compliance) |
| Heat-map / escalations | HTML artifact |
| Portfolio view | HTML artifact (numbers) + chat (narrative) |

HTML/artifact outputs carry the header short-form disclaimer (output-discipline §1).

---

## When new patterns surface

- New error / access pattern → `shared/error-handling.md`
- New data-source routing → `shared/source-routing.md`
- New cross-skill boundary → `shared/redirect.md`
- New scoring edge case → `references/scoring.md`
- New field definition → `references/field-registry.md`
- New user phrasing → add to the routing table above

---

*2026.06.13 — Migrated onto the Aegis shared layer (parity with Oracle). Output discipline + footer delegated to shared/output-discipline.md (removed the locally-defined footer/output-mechanics block; Radar takes the LEAN footer variant + HTML header short-form). Cross-skill boundaries delegated to shared/redirect.md (removed the hardcoded "Does NOT do / use X" list). No-access / unresolved identity → error-handling §0 escalate. Shared modules cited with raw URLs in the loading table (identity · source-routing · error-handling · output-discipline · redirect). Dual identity model kept explicit: references/identity.md (CSM/AE persona, Primary_Territory_Owner__gc) + shared/identity.md (leader tiers). R/E scoring + tech-touch logic untouched. `update` capability descoped (never built; write-adjacent, Phase 1 is read-only) — deferred to backlog. Single-fact reroute: Ask Gainsight retired → Gainsight connector directly.*
*2026-06-03 (v2.0) — Expanded from renewal/expansion-only (v1.3) to full operational workflow. Added my-plate, my-gaps, compliance, heat-map, portfolio, update from Sentinel v3.7.*
