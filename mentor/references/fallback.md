# Mentor — Fallback (offline / degraded mode)
# Aegis stack 2026.06.13
# Bundled (local). Loaded by SKILL.md ONLY when the hosted protocol.md fetch fails.
#
# Fail safe when Mentor can't reach its hosted instructions. Does NOT reconstruct routing or
# capability logic from memory — routing is hosted by design.

---

## When this loads

SKILL.md tried to fetch `mentor/protocol.md` and the fetch failed (network, repo unreachable, GitHub
outage). Mentor has no live routing. Degrade honestly.

## Behavior

1. **Tell the user plainly, once:**
   > *"Mentor couldn't load its latest instructions from the server, so I can't build 1:1 prep safely
   > right now. Usually a network or GitHub hiccup, not your setup."*

2. **Do not run capabilities.** No 1:1 prep reconstructed from memory — a half-remembered coaching
   brief about a real person is worse than none.

3. **Offer recovery:**
   > *"Try again in a minute. If it keeps failing, drop a note in #help-gainsight (CKBKTL1PD) and CS
   > Ops can check for a wider issue."*

4. **No writes, ever, in this mode.**

---

*2026.06.13 — Created with Mentor. Safe degraded-mode behavior: honest failure + recovery path, no reconstructed routing, no writes.*
