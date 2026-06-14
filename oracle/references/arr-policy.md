# Oracle — ARR Policy
# Aegis stack 2026.06.13
# Bundled (local). Load when about to render any ARR figure in any capability.

Canonical ARR render rules.

---

## Unrestricted — full figures, no ownership check

**`brief`:** ARR unrestricted for all users and all tiers (CSM, CSE, FLL, Not-CS). Render the exact ARR figure regardless of who pulls the brief. Single-account lookups don't expose portfolio-level data. No ownership check, no band fallback.

**`report`:** inherits the brief's render — ARR unrestricted (single-account brief output).

> Quick factual ARR lookups are served by the Gainsight connector directly (unrestricted, single-account, SFDC-equivalent). Team/portfolio ARR views are handled by Radar under its own ownership rules. That policy lives in Radar.

---

## Restricted — ownership check required

Applies to: `plan`.

Before rendering ARR $ or renewal $ in a success plan draft:

1. Verify `requester.Gsid == company.Csm` OR `requester.Gsid IN recursive_managers(company.Csm)`
2. If yes → render full ARR figure
3. If no → render `ARR_Band__gc` only, label `[restricted]`

This runs at render time on every output surfacing ARR — not a policy reminder, an enforcement step.

**ARR Band label format (live tenant, verified 2026-05-04):**
```
$10k or less | $10k - $50k | $50k - $100k | $100k - $250k | $250k or more
```
Use these exact strings when rendering ARR Band. Do not use the old format (`<$10k`, `$250k+`).

---

## Confidentiality rules (all capabilities)

- Never log customer name + ARR pairing to memory
- Never include ARR figures in external-facing content (LinkedIn drafts, Slack to external channels, email drafts)
- Pre-public roadmap mentioned in CTAs or timeline → mark "internal only" when surfaced
- Individual CSM performance comparisons → do not include ARR figures (team coaching output lives in Radar — that policy travels with it)

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
