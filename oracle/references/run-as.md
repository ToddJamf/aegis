# Oracle — CS Ops Run-As
# Aegis stack 2026.06.13
# Bundled (local). Load when `oracle as [email]` syntax is detected.
#
# Lets CS Ops view Oracle through another user's identity + scope for debugging, auditing, and
# operational analysis. A verified second identity path — not an exception to the immutable
# identification rule, but a CS-Ops-authenticated override.

---

## Syntax

```
oracle as [email]          → run the full skill as that user's identity + scope
oracle as [email] [slug]   → run a specific capability as that user (e.g. oracle as csm@example.com brief)
oracle as off              → clear override, return to your own identity
```

The `[email]` you type is the **target** you want to view as — resolved live against Gainsight at
invocation. It is runtime input, never stored.

---

## Authorization — single live check

Query `gsuser` for the requester. Authorize **only** if `Employee_Type__gc` resolves to the CS
Operations role value.

```python
authorized = (requester.Employee_Type__gc == <CS Operations value — see field-registry.md>)
```

No hardcoded allowlist. Authorization lives in Gainsight, not in this file — if someone who should
have access doesn't, the fix is to set their `Employee_Type__gc` in Gainsight, not to enumerate
people here. (A legacy hardcoded email list was removed 2026-06-13 — it was an artifact of an earlier
access-control approach, and employee emails don't belong in a public repo.)

If the check fails → hard fail:
> *"'oracle as' is restricted to CS Operations. Your Gainsight role doesn't authorize it."*

---

## What it does

- Resolves the target user's full identity from Gainsight (tier, book scope, manager chain, teamGsids).
- Runs any capability exactly as if that user invoked it — including tier-gated views.
- ARR is unrestricted for CS Ops — full figures regardless of ownership.
- Caches override as `runAsIdentity`, separate from `identity` (requester's own record preserved).

**Note — the seat is still the seat.** Reads execute as the *connected* Gainsight seat with its
server-side permissions; run-as changes the identity *model* Oracle reasons with, not the underlying
data authorization. It's a read-only courtesy lens, consistent with the write-gateway stance.

---

## Output label — every output, no exceptions

```
⚙ CS Ops — viewing as [target name] ([target email])
```

Cannot be suppressed. Applies to every render while run-as is active. (Target name/email here are
runtime display of who you're viewing as — not stored data.)

---

## What it does NOT do

- No writes — still read-only.
- Does not suppress the label.
- Does not cascade: `oracle as [another_email]` without `oracle as off` first → refuse:
  *"Already viewing as [current target]. Run 'oracle as off' first."*
- Does not persist between sessions.

---

## Target not found

After authorization passes, run the gsuser query for the target email. 0 results →
*"Couldn't find [email] in Gainsight. Check the address. If the account exists but is inactive, contact CS Ops."*
Do not modify `runAsIdentity`; do not proceed.

---

## Session lifecycle

- `oracle as [email]` → sets `runAsIdentity`, labels all output.
- `oracle as off` → clears it, returns to requester's identity. Render:
  ```
  ⚙ CS Ops — run-as cleared. Viewing as [requester name] ([requester email]).
  ```
- Session end → `runAsIdentity` auto-cleared.

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle. Removed the hardcoded CS-Ops email allowlist (employee-email PII; artifact of an old access-control stance now dropped) — authorization is the single live `Employee_Type__gc = CS Operations` check. Sentinel → Oracle rebrand. CalVer.*
