# Radar — Fallback (offline / degraded mode)
# Aegis stack 2026.06.13
# Bundled (local). Loaded by SKILL.md ONLY when the hosted protocol.md fetch fails.
#
# Purpose: fail safe when Radar can't reach its hosted instructions. This file does NOT
# reconstruct routing or capability logic — routing is hosted by design, and faking it risks
# the divergence the original audit flagged. It handles the failure cleanly and points the user
# to a fix.

---

## When this loads

SKILL.md tried to fetch `radar/protocol.md` from GitHub and the fetch failed (network down, repo
unreachable, GitHub outage). Radar has no live routing. Do not improvise capability logic from
memory — degrade honestly instead.

---

## Behavior

1. **Tell the user plainly, once:**
   > *"Radar couldn't load its latest instructions from the server, so I can't run a renewal or
   > expansion view safely right now. This is usually a network or GitHub hiccup, not your setup."*

2. **Do not run capabilities.** No renewal triage, no expansion pipeline, no plate view, no
   compliance or heat-map reconstructed from memory. A half-remembered scoring pass is worse than
   none — it can be wrong and look authoritative, and Radar drives renewal action.

3. **Offer the recovery paths:**
   > *"Try again in a minute — it's usually transient. If it keeps failing, drop a note in
   > #help-gainsight (channel CKBKTL1PD) and CS Ops can check whether it's a wider issue."*

4. **No writes, ever, in this mode** — same hard rail as normal operation. No account updates,
   no Timeline entries, no CTAs.

5. **If a bundled reference is genuinely self-contained and the user's need is met by it alone**
   (e.g. a definitional question answerable from a bundled file already in the package), it's fine
   to answer from the bundled file — but say it's a local definition, not a live lookup.

---

## What this is not

Not a mini-protocol. Not a place to add routing tables or rebuild the scoring formulas. If Radar
needs richer offline behavior later, that's a deliberate design decision — not something to grow
here by accretion. Fail safe, point to the fix, stop.

---

*2026.06.13 — Created. Radar's safe degraded-mode behavior, mirroring oracle/references/fallback.md: honest failure + recovery paths (#help-gainsight, channel CKBKTL1PD), no reconstructed routing or scoring, no writes. Radar-flavored copy.*
