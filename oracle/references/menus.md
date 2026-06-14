# Oracle — Menus (tier-aware)
# Aegis stack 2026.06.13
# Bundled (local). Load when rendering the `menu` capability.
#
# Oracle is nearly tier-flat after the carve-out: everyone gets brief + glossary + help + report;
# only `plan` is CS-team. So two variants — CS team and Open. 1:1 prep moved to Mentor; operational
# views moved to Radar; quick single-fact lookups go to the Gainsight connector directly. The menu advertises ONLY what Oracle does and
# points elsewhere for the rest (never advertise a capability that moved — that's a dead-end).

---

## CS-team menu (CSM / FLL / Segment Leader / SR Leadership / Department Head)

```
🔮 Oracle — account meeting prep
[your name]

  brief      Brief me on [account]
             → Health + trend, renewal, CTAs, success plans, products, ranked activity, Staircase, prep

  plan       Draft a success plan for [account]
             → Signal-driven draft, paste-ready for Gainsight entry

  glossary   What is a [term]?
  report     Flag a wrong answer (`oracle report`)
  help       Docs and how Oracle works

Looking for something else?
  • 1:1 / prep for a colleague → Mentor      • What's on my plate / renewals / team views → Radar
  • Quick fact ("when does Acme renew?") → ask the Gainsight connector directly   • ARR analytics → ask-snowflake-analyst

Just ask — natural language works.
  • "Brief me on Acme"   • "I have a call with Acme"   • "Draft a success plan for Acme"   • "What is a CTA?"
```

---

## Open menu (authenticated, no CS role)

```
🔮 Oracle — account meeting prep

  brief      Brief me on [account]
             → Full brief: health, renewal, open items, products, recent activity, prep
             → Renders what your Gainsight access allows

  glossary   What is a [term]?
  report     Flag a wrong answer (`oracle report`)
  help       Docs and how Oracle works

Looking for something else?
  • Quick fact ("who's the CSM on Acme?") → ask the Gainsight connector directly

Just ask — natural language works.
  • "Brief me on Acme"   • "I have a call with Acme"   • "What is a CTA?"
```

Success-plan drafting is a CS-team capability — not shown here. If an Open user asks for one, the
capability returns the scope note (see capability-plan.md), not this menu.

---

*2026.06.13 — Rewritten for Oracle. Collapsed five Sentinel variants (CSM/FLL/Segment/SR/Not-CS) to two (CS team / Open) — the tier variants mostly existed to gate 1:1, which moved to Mentor. Removed every 1:1 entry and the operational entries (my-plate, my-gaps, update, compliance, heat-map, portfolio) that belong to Radar — replaced with redirect pointers. Sentinel → Oracle rebrand. Example names genericized to Acme per repo confidentiality rule.*
