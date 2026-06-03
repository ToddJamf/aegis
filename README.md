# Aegis

Hosted logic layer for Jamf CS Ops Claude skills. Skills contain a thin bootstrap that fetches actual instructions from this repo at runtime. Update a file here → all users pick it up automatically, no reinstall required.

---

## Structure

```
aegis/
  shared/
    identity.md              ← auth / user identity resolution (shared across all skills)
    error-handling.md        ← error patterns, safe call discipline (shared across all skills)
  sentinel/
    protocol.md              ← Sentinel router + orchestration (always fetched)
    capability-brief.md      ← Cap 3: pre-call brief
    capability-top-accounts.md      ← Cap 4: top accounts to work (CSM)
    capability-activity-gap-csm.md  ← Cap 5: activity gap scan (CSM)
    capability-leader-activity-gap.md   ← Cap 6: activity gap report (Leader)
    capability-leader-escalations.md    ← Cap 7: top accounts to escalate (Leader)
    capability-success-plan.md          ← Cap 10: draft success plan
```

**Bundled in each skill package (not hosted here — stable reference data):**
- `field-registry.md` — Gainsight field names
- `terminology-cs.md` / `terminology-public.md` — glossary
- `efficiency.md` — query design patterns
- `setup.md`, `test-mode.md`, `overview.md`

---

## Update workflow

1. Edit any file in this repo
2. Commit + push to `main`
3. All users pick up the change on next skill activation — no reinstall

---

## Raw fetch URLs

| File | URL |
|------|-----|
| `shared/identity.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/identity.md` |
| `shared/error-handling.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/shared/error-handling.md` |
| `sentinel/protocol.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/protocol.md` |
| `sentinel/capability-brief.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-brief.md` |
| `sentinel/capability-top-accounts.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-top-accounts.md` |
| `sentinel/capability-activity-gap-csm.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-activity-gap-csm.md` |
| `sentinel/capability-leader-activity-gap.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-leader-activity-gap.md` |
| `sentinel/capability-leader-escalations.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-leader-escalations.md` |
| `sentinel/capability-success-plan.md` | `https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-success-plan.md` |

---

## Skill bootstrap (thin SKILL.md)

The installed Sentinel `SKILL.md` contains only:
1. Frontmatter (name, description for skill triggering)
2. One instruction: fetch `sentinel/protocol.md` and execute it

The protocol then fetches shared and capability files on demand.

---

*Aegis v2.0 — June 2026 | Todd Massey, CS Ops*
