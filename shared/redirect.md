# Shared — Redirect (cross-skill dispatch)
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/redirect.md
#
# Shared across: Oracle, Mentor, Radar, Pulse
# The stack's one map of who-owns-what. A skill that decides a request isn't its job consults
# this to find the right skill and hand off cleanly. Defined once here, not re-stated per skill.
# Message rules: shared/output-discipline.md.

---

## Why this is shared

Each skill should know exactly one thing about its siblings: nothing. A skill knows **its own** capabilities — "is this request mine?" — and when the answer is no, it asks this map "then whose?" That keeps every skill ignorant of the others, so adding **Mentor** (or any future skill) means editing this one file, not patching five skills' boundary copy.

This replaces the duplicated "Do NOT use for X — use Y" lines currently repeated in every `SKILL.md` description and the README scope table. Those become pointers here.

Coherence with `source-routing.md`: that file routes **data reads** (which tool answers a data question). This file routes **intent handoffs** (which skill owns a request the current skill doesn't). A skill recognizing its *own* capabilities stays local to that skill; the cross-skill map is shared.

---

## The dispatch flow

```
1. Skill matches the request against ITS OWN capabilities.
      owned?  → handle it. Stop. (No redirect lookup on in-scope work.)
2. Not owned → consult the territory map below: whose territory does this fall in?
3. Match → render the redirect message (one pattern, below), carrying the user's ask over.
4. No match → unowned fallback (below). Never guess a skill.
5. Could match two (name collision) → ask ONE disambiguating question before routing.
```

The lookup only fires on out-of-scope requests — in-scope work never pays the cost.

---

## Territory map

| Skill | Owns | Trigger surface (illustrative) |
|-------|------|-------------------------------|
| **Oracle** | Meeting prep on an **account** | "brief me on [account]", "I have a call with [account]", "draft a success plan for [account]" |
| **Mentor** | Prep on a **person** (1:1s, any direction) | "prep me for [colleague]", "1:1 with [name]", "prep for my 1:1 with [manager]" |
| **Radar** | Operational CS workflow | "what's renewing", "expansion pipeline", "what's on my plate", "activity gap", "team compliance", "heat map", "portfolio", "what should I escalate" |
| **Pulse** | Community intelligence | "community dashboard", "dormant accounts", "unanswered posts", "Heroes tracker" |
| **ask-snowflake-analyst** | ARR / license analytics, Snowflake | ARR trends, license counts, anything Snowflake-backed |

**Quick single-fact lookups** ("when does Acme renew?", "who's the CSM on Acme?", "health score for Acme?") are served by the Gainsight connector directly — they are no longer a skill. Glossary ("what is a CTA?", "explain Customer Category") is owned by **Oracle**. For a single fact, point the user at the Gainsight connector (or offer Oracle for a full brief); for a term, route to Oracle.

The boundary that does the most work: **account vs. person.** Names a customer → Oracle. Names a colleague → Mentor.

---

## The redirect message — one pattern, every skill

```
[what they asked] is [target]'s job, not [current skill]'s — [one-line why].
Ask [target]: "[restated ask, rephrased for the target]"
[if-missing tail — see below]
```

Three rules, all non-negotiable:

1. **Name the target skill.** Never a bare refusal. A user who hits "I can't do that" twice files a defect.
2. **Carry the ask.** Restate their request rephrased for the target so they never retype. "Prep me for Sarah" → *"Sarah's a colleague — that's Mentor's prep, not Oracle's. Ask Mentor: '1:1 prep for Sarah'."*
3. **Don't assume they have it** — see below.

Tone follows output-discipline — plain, internal, a little warmth. This is a helpful hand-off, not an error.

---

## When the target skill isn't installed

A skill **cannot detect what else the user has installed** — each Aegis skill is independent and has no visibility into its siblings. So a redirect can't promise the target is there. Write every redirect to work both ways: the user either already has the target (they just invoke it) or they don't (they need the path to add it). One line covers both:

```
Ask [target]: "[restated ask]"  ·  Don't have [target] yet? Add it from the Aegis skill library (or ping #help-gainsight).
```

If they have it, they ignore the tail. If they don't, they know exactly how to get it. Never leave a redirect that dead-ends at a skill the user can't reach — that's the same dead-end failure §0 exists to prevent, just one layer up.

**Single-skill reality:** plenty of users will run Oracle alone, no Radar or Mentor. The redirect is still correct and still useful for them — it tells them the capability exists and how to unlock it, instead of pretending Oracle does it or refusing flat.

---

## Name-collision disambiguation

When a request could belong to two skills because a name is ambiguous — "prep me for Morgan" where Morgan is both a colleague and an account — **do not guess.** Ask one line:

> "Morgan the account, or Morgan a colleague? (Account → I've got it; colleague → that's Mentor.)"

Cache the answer for the session. One question, then route.

---

## Unowned fallback

Request matches no skill's territory → don't invent a destination.

> "That's outside what the CS skills cover. Closest I've got is [nearest skill, if any] — want that instead?"

Keep this distinct from `error-handling.md` §0: **§0 is access** (you're blocked from data you should reach), **this is scope** (no CS skill does this). Don't route a scope miss to the access-escalation block — they're different problems and a confused user gets a worse answer.

---

*2026.06.13 — Created. Shared cross-skill dispatch: territory map + one redirect-message pattern + collision disambiguation + unowned fallback. De-duplicates per-skill scope-boundary copy. Anchors the Oracle→Mentor carve-out redirect. Distinct from source-routing (data) and §0 (access). Added partial-install handling — skills can't see sibling installs, so every redirect carries the "don't have it? add it" path; single-skill (Oracle-only) users get a correct, useful redirect.*
