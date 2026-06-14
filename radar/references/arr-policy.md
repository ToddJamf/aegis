# Radar — ARR Display Policy
# Aegis stack 2026.06.13
# Bundled (local). Load whenever a view renders an ARR figure (renewal lane, expansion lane,
# list caps, leader/portfolio rollups, stats-bar "ARR at stake").

Radar renders ARR across LIST views and leader views, so its disclosure rule is scope-aware —
unlike a single-account brief. This file is the one place that decides whether a row shows the
exact `$` figure or falls back to a band label. Every render path defers here.

---

## The rule (precedence order)

Evaluate top-down; first match wins.

| # | Scope of the row being rendered | What to show |
|---|---|---|
| 1 | **CS Ops run-as** (operator-scoped session) | Full ARR `$` figure — unrestricted, all rows |
| 2 | **Own book / leader scope** — account is inside the requester's owned scope (CSM book via `Csm`, AE territory via `Primary_Territory_Owner__gc`, or a leader's down-tree per shared/identity.md) | Full ARR `$` figure |
| 3 | **Out of owned scope** — account surfaced in a list cap (e.g. "and N more", cross-book heads-up, a portfolio row outside the leader's tree) | `ARR_Band__gc` **label**, not the exact figure |

"Owned scope" = the same scope `identity.book_field` / the leader cache already define
(references/identity.md, shared/identity.md). Radar does not introduce a new scope concept here —
it reuses persona/leader scope and applies a display rule on top.

---

## Band fallback — use the picklist label, not a derived bucket

When rule 3 applies, render `account.ARR_Band__gc_PicklistLabel` verbatim. Do not compute a band
from `Arr` and do not round the figure — the point is to not disclose the exact dollar amount
outside owned scope. The picklist is defined in `references/field-registry.md`:

```
ARR_Band__gc labels (Jamf tenant):
  "$10k or less"  ·  "$10k - $50k"  ·  "$50k - $100k"  ·  "$100k - $250k"  ·  "$250k or more"
```

(GSIDs for each band are in field-registry.md under `PICKLISTS[("company","ARR_Band__gc")]` —
config records, fine to carry.) If `ARR_Band__gc` is null on an out-of-scope row, render `—`,
never the raw `$` figure.

---

## Render-path hook

`arrDisplay` in `references/artifact.md` is the single implementation point:

```js
// applies this policy
if (!inScope(account)) return account.ARR_Band__gc_PicklistLabel || '—';  // rule 3
var arr = account.Available_to_Renew__gc || account.Arr;
if (!arr) return account.ARR_Band__gc_PicklistLabel || '';
return '$' + (arr >= 1000 ? (arr/1000).toFixed(1) + 'K' : arr);           // rules 1 & 2
```

`inScope(account)` = CS Ops run-as → always true; otherwise true when the account's owner field
matches the cached identity scope. Stats-bar "ARR at stake" sums **only in-scope** accounts at full
figure; it never sums out-of-scope rows (they have no figure to sum).

---

## Confidentiality hard rails (non-negotiable)

- **Never log customer-name + ARR to memory** — not even in-scope, full-figure rows. ARR figures
  are sensitive; pause and ask before any memory write that pairs an account with a dollar amount.
- **Never put ARR (figure or band) in external-facing content** — LinkedIn drafts, anything that
  leaves the tenant. Internal artifact render only.
- The band-fallback rule is a disclosure floor, not a license — being inside owned scope permits
  display, it does not permit logging or export.

---

*2026.06.13 — Created (net-new, Radar-specific — NOT ported from Oracle's single-account arr-policy). Encodes scope-aware list/leader ARR disclosure: CS Ops run-as + own/leader scope → full `$`; out-of-scope list caps → ARR_Band__gc label; null band → `—`. References the ARR_Band picklist in field-registry.md (labels reproduced, GSIDs left there). Memory + external-content hard rails carried from the OS confidentiality posture. No customer data.*
