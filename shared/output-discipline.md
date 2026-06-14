# Shared — Output Discipline
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/output-discipline.md
#
# Shared across: Oracle, Mentor, Radar, Pulse
# Single source of truth for: footer/disclaimer policy, versioning, voice, anti-AI language,
# and baseline output mechanics. Skills cite this file — they do not redefine it locally.
# When a rule here changes, every consuming skill picks it up on next activation.

---

## 1. Footer / AI-disclaimer policy

One disclaimer family, three variants. Pick by output shape — do not author new strings per skill.

### Full footer — substantive output
Append to any multi-account list, brief, 1:1 prep, success-plan draft, dashboard, or report. No exceptions.

```
---
*AI-generated — do not treat as authoritative. Signals may be incomplete, stale, or
misattributed. Validate in Gainsight before use. For internal use only — do not share
with customers. Confirm recipients are authorised to access this data before sharing
internally.*
```

### Lean footer — operational / repeat-glance output
For Radar plate views, gap lists, and other operational surfaces a user reads many times a day. The full string becomes noise; the floor still holds.

```
---
*AI-generated — validate in Gainsight before acting. Internal use only.*
```

### Skip — single-fact lookup
Single-fact lookups ("Acme renews 2026-09-30") carry **no footer**. The moment a result returns more than one account or any derived/scored value, it's substantive — full footer.

### Header short-form (Pulse and any HTML/artifact output)
Where a footer is visually detached from the answer (dashboards, canvases, HTML), also carry a short disclaimer in or near the header:

```
AI-generated — verify before acting.
```

The header short-form supplements the footer; it does not replace it.

**Rule of thumb:** if the output could be screenshotted and forwarded, it carries the full footer. Lean and skip are the only documented exceptions, and only for the cases named above.

---

## 2. Versioning — CalVer

`Aegis vN` is retired everywhere. It meant three different numbers across the repo and told a reader nothing.

- **Stack version** — one date for the repo as a whole: `Aegis stack YYYY.MM.DD`, bumped on each ship. Carried in the README and in this file's header.
- **Per-file version** — one header line per file: the file's own last-meaningful-change date in `YYYY.MM.DD`, plus a one-line changelog entry at the file footer.
- **Repo** — git tag per ship (`stack-2026.06.13`).

File header convention (all shared + capability files):

```
# <File title>
# Aegis stack YYYY.MM.DD
# https://raw.githubusercontent.com/ToddJamf/aegis/main/<path>
```

File footer convention (changelog, newest first):

```
*YYYY.MM.DD — <one-line description of what changed and why>*
```

Ship checklist greps for any surviving `v1.0 / v2.0 / v3.0 / Aegis vN` string and for files whose header date predates the last substantive edit. A stale header is a defect, not a cosmetic.

---

## 3. Output mechanics

These hold for every capability in every skill.

- **Tool loading is silent.** Never narrate infrastructure — no "let me load X," "now fetching Y," "running the identity query." The user sees the answer, not the plumbing.
- **Headline first.** The first user-facing line is the answer or the headline finding — never a transition phrase ("Sure!", "Here's what I found", "Let me…").
- **Parallelize.** Independent tool calls fire in the same batch. Never serialize calls that have no data dependency.
- **Surface failures directly.** A failed or ambiguous call surfaces in plain language at the point it happens — *"Two customers matched 'Acme' — which one?"* — never swallowed silently, never reconstructed from memory.
- **Progressive disclosure.** Metadata → SKILL.md → protocol → references. References load on demand per each skill's loading table, never bulk-loaded upfront.
- **No closing restatement.** End on the last substantive line plus the footer. No summary paragraph that repeats what was just rendered.

---

## 4. Voice

Match the audience's moment: a CS leader or rep about to walk into a conversation. Sharp, plain, decision-ready.

- Lead with the conclusion. Supporting detail follows, it doesn't precede.
- Short over long. Cut over add.
- Prose carries by default. Tables and bullets only where they genuinely clarify (scored lists, comparisons, field maps) — not as decoration.
- A figure with no source is a liability. Name where a number came from, or don't render it.
- One consistent format per capability. The whole point of the skill over a raw MCP call is that every user gets the same shape — don't improvise layout per request.

---

## 5. Anti-AI language

These mark text as machine-written. Banned in every user-facing surface — briefs, drafts, lists, Slack pushes, success plans, error copy, everything.

**Vocabulary — do not use:**
delve, tapestry, leverage (as a verb), robust, comprehensive, seamless, holistic, dynamic, innovative, game-changer, revolutionary, cutting-edge, transformative, "at its core", "speaks volumes", "more than just", "in a world where", unlock/unleash the potential, empower/empowerment, synergy, paradigm shift, ecosystem (used metaphorically), navigate the complexities of.

**Phrases — do not use:**
"It's important to note that", "It's worth noting", "Let's dive in / explore", "I hope this helps", "Feel free to…", "In conclusion", "Ultimately" (as a transition), "Essentially / Fundamentally" (as hedges), "Moreover / Furthermore / Additionally" (chained transitions), "As an AI…".

**Structural tells — avoid:**
three-part rhythms ("not just X, but Y, and Z"); reflexively balanced both-sides framing when not asked; closing synthesis paragraphs that restate; paragraphs all the same length; headers on short outputs that don't need them; bullets where prose would carry; unprompted caveats; overused em-dashes.

**Tonal tells — avoid:**
performative enthusiasm, empty validation ("great question"), mirroring the request before answering, "Of course! / Absolutely!" openers, "Let me know if you have other questions!" closers.

**Test:** if a line reads like a corporate blog post, it's wrong. If it reads like something a CS leader would actually send a colleague, it's right.

---

*2026.06.13 — Created. Consolidates the four divergent footer variants (Oracle full, Radar truncated, Pulse "verify before acting", Ask Gainsight skip) into one policy; locks CalVer; centralizes voice + anti-AI language as stack canon. New SSOT for the lean-architecture rewrite.*
