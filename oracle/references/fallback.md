# Oracle — Fallback (offline / degraded mode)
# Aegis stack 2026.06.13
# Bundled (local). Loaded by SKILL.md ONLY when the hosted protocol.md fetch fails.
#
# Purpose: fail safe when Oracle can't reach its hosted instructions. This file does NOT
# reconstruct routing or capability logic — routing is hosted by design, and faking it risks
# the divergence the original audit flagged. It handles the failure cleanly and points the user
# to a fix.

---

## When this loads

SKILL.md tried to fetch `oracle/protocol.md` from GitHub and the fetch failed (network down, repo
unreachable, GitHub outage). Oracle has no live routing. Do not improvise capability logic from
memory — degrade honestly instead.

---

## Behavior

1. **Tell the user plainly, once:**
   > *"Oracle couldn't load its latest instructions from the server, so I can't run a full brief
   > safely right now. This is usually a network or GitHub hiccup, not your setup."*

2. **Do not run capabilities.** No brief, no plan, no glossary reconstructed from memory. A
   half-remembered brief is worse than no brief — it can be wrong and look authoritative.

3. **Offer the recovery paths:**
   > *"Try again in a minute — it's usually transient. If it keeps failing, drop a note in
   > #help-gainsight (channel CKBKTL1PD) and CS Ops can check whether it's a wider issue."*

4. **No writes, ever, in this mode** — same hard rail as normal operation.

5. **If a bundled reference is genuinely self-contained and the user's need is met by it alone**
   (e.g. a pure terminology question answerable from `glossary.md` already in the package), it's
   fine to answer from the bundled file — but say it's a local definition, not a live lookup.

---

## What this is not

Not a mini-protocol. Not a place to add routing tables. If Oracle needs richer offline behavior
later, that's a deliberate design decision — not something to grow here by accretion. Fail safe,
point to the fix, stop.

---

*2026.06.13 — Created. Oracle never had a real fallback.md (phantom reference flagged in the hard audit). Authored as safe degraded-mode behavior: honest failure + recovery paths (#help-gainsight), no reconstructed routing, no writes.*
