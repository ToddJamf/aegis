# Oracle — Test Mode
# Aegis stack 2026.06.13
# Bundled (local). Load when a user activates test mode via `oracle test` or `oracle test as [email]`.

---

## Purpose

Bypasses the identity confirmation prompt so QA testers can iterate quickly without
answering "Running as X — is that right?" on every session. Identity still resolves
from Gainsight — real data, real tier detection, no synthetic mocks.

---

## Activation

| Command | Behavior |
|---------|----------|
| `oracle test` | Skip confirmation. Use session email for identity resolution. |
| `oracle test as [email]` | Skip confirmation. Use specified email for identity resolution. |
| `oracle test off` | Deactivate test mode. Resume normal confirmation behavior. |

Test mode persists for the session once activated.

---

## Behavior in test mode

1. Run Step 0 (MCP inventory) normally.
2. Run Step 1 identity query normally — resolve from Gainsight using the specified email.
3. Skip the confirmation prompt. Set `verified_at = "TEST_MODE"`.
4. Prepend `🧪 TEST MODE` to every user-facing output for the session. No exceptions — menu renders, briefs, plans, glossary lookups. All of them.
5. All other skill logic runs exactly as in production — tier gates, territory model, ARR render check, write block, MCP degraded mode. Test mode bypasses the confirmation only.

---

## "test as" — admin override list

Some users have unrestricted `test as` access — they can proxy as any Gainsight user regardless of org chart. This is for skill builders and CS Ops admins who need to QA the full CS org's experience.

**Admin list (hardcoded here — edit to add/remove the authorized admin emails):**

```
admin@example.com
```

**Authorization flow for `oracle test as [email]`:**

> **Bootstrap guard:** If `oracle test as [email]` is the first message of the session and requester identity is not yet cached, run Step 0 and Step 1 identity resolution for the *requester* first, then proceed with authorization below. Do not attempt the admin check or org chart gate without a resolved requester identity.

1. Resolve the requester's identity from Gainsight (Step 1).
2. **Check admin list first.** If `requester.Email IN admin_list` → skip org chart gate entirely. Go to step 4.
3. If not admin → verify `target.Gsid IN requester.teamGsids` (recursive org walk). If not in org → refuse:

```
🧪 TEST mode restricted to your org chart.
[email] is not in your reporting chain — test as someone who reports to you.
```

4. Resolve the target email from Gainsight.
5. If target not found in Gainsight:
```
🧪 TEST MODE — [email] not found in Gainsight. Check the email or try a different one.
```
6. Activate test mode as the target user. Set `verified_at = "TEST_MODE (admin proxy: [requester.Email])"`.

**Non-admin Leaders** can still use `test as` scoped to their own org chart. Plain `oracle test` (own identity) is available to all tiers. CSMs cannot use `test as` unless on the admin list.

**Use cases for admins:**

- QA a CSM's brief view: `oracle test as csm@example.com`
- QA a Leader's brief access: `oracle test as fll@example.com`
- Test cross-region territory flag from another team's perspective
- Validate a bug report from a specific user: `oracle test as user-who-reported-bug@example.com`

---

## What test mode does NOT bypass

- MCP availability check (Step 0) — connectors must still be live
- Tier gates — Not-CS users still get Glossary only
- ARR confidentiality render check — ARR $ vs ARR Band rules still apply
- Write block — skill is still read-only
- Territory soft flag — cross-region notice still appears (but labeled TEST MODE)

---

## Turning it off

`oracle test off` → resume normal behavior. Next data operation will prompt for
identity confirmation as usual.

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
