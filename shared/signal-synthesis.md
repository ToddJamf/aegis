# Aegis Shared — Signal Synthesis
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/signal-synthesis.md
#
# Sentinel trunk governance. All Sentinel-derived skills (Oracle, Radar, Pulse, Mentor)
# apply this framework when triangulating data signals into synthesized output.
# This is reasoning governance — it governs HOW skills interpret data, not what they display.
# Output format is determined by each capability module. Labels never appear in user output.

---

## The DO / HEAR / SAY framework

Every judgment in the Sentinel stack is evaluated against three witnesses:

| Witness | Source | What it reads |
|---------|--------|---------------|
| **SAY** | Gainsight (scorecard, sentiment, CTAs, timeline notes) | What's recorded — the official CS picture |
| **HEAR** | Staircase AI (comms signal, sentiment trend, champion signal, engagement) | What the relationship sounds like |
| **DO** | Gainsight products (deployment%, last login, activity frequency, community) | What the customer actually does |

---

## The two-witness rule

**Two agree → trust it.** Confirmed picture. Output reflects the consensus signal.

**Two disagree → that is the insight.** The divergence between recorded truth and behavioral/relationship truth is the highest-value output the skill can produce. Surface it explicitly. Don't smooth it over.

The three common divergence patterns:

| Pattern | SAY | HEAR or DO | Verdict |
|---------|-----|------------|---------|
| Green that is not green | Healthy / stable | Risk signals | SAY is optimistic — real picture is worse. Walk in knowing. |
| Shelfware hiding in plain sight | Healthy | Low deployment / zero login | Value hasn't landed. Renewal talk is about outcomes, not contract. |
| Quiet risk | No open CTAs, no flags | Declining engagement, tone shift | Nothing is officially wrong. Relationship is drifting anyway. |

---

## Graceful degradation

**Staircase not connected:** HEAR is unavailable. Run SAY / DO comparison only — two-witness check is still better than none. Note absence in output: "Staircase not connected — HEAR signal unavailable."

**No product data:** DO is unavailable. Run SAY / HEAR only. Note: "No product relationships — DO signal unavailable."

**Both unavailable:** SAY only. Render what's available. Flag explicitly — don't hallucinate a verdict.

---

## Output contract

**Labels never appear in user-facing output.** DO / HEAR / SAY is the reasoning layer, not the display layer.

The output of the synthesis is:
- A plain-language description of what the signals show
- An explicit call-out when two witnesses disagree (the insight)
- A concrete action or opening line grounded in the verdict

Skills decide their own output format (brief prose, triage flag, table row). This module governs only the reasoning — not the render.

---

## Application by skill

| Skill | Synthesis depth | Output format |
|-------|----------------|---------------|
| Oracle (capability-brief) | Full three-witness synthesis per account | Prose verdict + opening line in 🎯 Prep block |
| Radar | Triage-level flag per account row | ⚠️ Signal mismatch chip + one-line note |
| Pulse | Community DO signal vs. Gainsight SAY | Noted where community activity contradicts health record |
| Mentor | SAY (what's logged) vs. HEAR (how the CSM talks about the account) | Coaching observation, not verdict |

---

## Governance note

This framework is the Sentinel trunk DNA. It does not replace data fidelity or coverage obligations
(error-handling.md). It applies *after* data is fetched and verified — it is the synthesis step,
not the fetch step.

---

*2026.06.13 — Created as Aegis shared governance module. DO/HEAR/SAY triangulation framework extracted
from capability-brief.md Prep Synthesis section and elevated to trunk-level governance. Applies to
Oracle, Radar, Pulse, Mentor. Labels are reasoning-layer only — never rendered in user output.*
