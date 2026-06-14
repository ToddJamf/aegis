# Oracle — Synonym Map (user phrasing → field)
# Aegis stack 2026.06.13
# Bundled (local). Load for query translation when building a brief or success plan.

Lightweight routing file. For full CS term definitions, cadence thresholds, and product detail — load `terminology-cs.md`.

Oracle's scope is meeting prep: pre-call briefs (`brief`) and success plan drafts (`plan`), plus glossary. Phrases that belong to other skills are noted with a → pointer; Oracle does not service them.

---

## In-scope (brief / plan)

| User says | Maps to |
|-----------|---------|
| "Cockpit" | Open CTAs (`fetch_cta_list`, `IsClosed=false`) |
| "Plays" | CTAs, often playbook-driven |
| "Tasks under [CTA]" | `cs_task` records linked to that CTA |
| "Spotlight / Retain / Grow / React" (as account category) | `Customer_Category__gc = <value>` |
| "Health" / "scorecard" | `ask_scorecard` |
| "Sentiment" (alone) | CSM Sentiment A/B/C/D/F |
| "Risk signals" | Pattern A (single account): `staircase_account_lookup` → `staircase_analyze_account`. See field-registry.md. |
| "Activity" / "what's happened" | `fetch_timeline_activity_list` |
| "Renewal status" (single account, for brief context) | `Renewal_Status__gc` |
| "EBR" / "QBR" | Business Review meeting |
| "Config Review" | Tracked separately from BR — admin-level review |

---

## Out of scope — route elsewhere

These phrasings are NOT Oracle capabilities. Hand off, don't map to a query.

| User says | Belongs to |
|-----------|-----------|
| "When does Acme renew?" / "who is the CSM on Acme?" / "health score for Acme?" (quick factual lookup) | → Gainsight connector directly (or Oracle for a full brief) |
| "What's on my plate" / "top accounts to work this week" | → Radar |
| "Who haven't I touched" / "activity gaps" / "engagement gap" | → Radar |
| "Team compliance" / "who needs coaching" / "team engagement" | → Radar |
| "Top accounts to escalate" / "what should I escalate" / "team risk" / "heat map" | → Radar |
| "Portfolio view" / "segment story" / "what do I tell my boss" | → Radar |
| "Update for [account]" | → Radar |
| "1:1 with [name]" / "prep me for my 1:1" / person prep | → Mentor |

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
