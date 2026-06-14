# Oracle — Write Gateway
# Aegis stack 2026.06.13
# Bundled (local). Step 3 of Oracle's pipeline — every write-intent routes here before any write
# MCP tool is called. A structural chokepoint, not a policy reminder.
#
# Load when: any write-intent signal in the user's request.

---

## ⚠ Writes execute as the connected seat — read this first

The Gainsight MCP runs every call as the **authenticated connection**, with Gainsight's server-side
permissions. That has one sharp consequence for this gateway:

**`verify_downward` and the tier model are courtesy controls, not security controls.** A write
executes as whoever is connected, regardless of what Oracle's tier model resolved. Oracle's checks
decide *whether to offer* a write; they cannot *enforce* who it runs as — Gainsight does that, at the
seat level. This must stay true in every Phase 2 unlock: never describe a tier check as if it gates
write authority. The seat is the authority.

---

## Currently allowed writes

**Phase 1: none.** Oracle is read-only.

**System exemption — `report`:** `slack_send_message` to `CS_OPS_REPORT_CHANNEL` is exempt — it's a
system-integrity function (flagging a wrong answer to CS Ops), not a Gainsight data write. Phase 1,
that channel and purpose only. Any other Slack send stays blocked.

Phase 2 will unlock (list updated here when approved):
- `create_timeline_activity` — meeting notes (single account, CSM-scoped, explicit confirmation)
- `manage_success_plan_actions` — success plan create (single account, explicit confirmation)

---

## Scope check

Classify intent:
- **Read** → skip this file, proceed.
- **System-exempt write** → `report` → `CS_OPS_REPORT_CHANNEL` only → ALLOW, skip compliance check.
- **Write** → run compliance check.

Write-intent signals: "log this call", "add to timeline", "create a CTA", "update the plan", "write
to Gainsight", "send to Slack", "post this", or anything that mutates external data.

---

## Compliance check

```python
is_write_allowed(operation, requester, target):
  → if operation == "report_slack_send":   ALLOW (system exemption)
  → operation IN allowed_writes            (Phase 1: empty → always BLOCK)
  → requester authorized for target        (courtesy check — own book / FLL verify_downward;
                                            NOT a security boundary — see top of file)
  → target is a single account             (bulk = hard block)
  → Returns ALLOW or BLOCK
```

**If BLOCK** → route to the Gainsight UI with paste-ready output:
*"Gainsight writes aren't live in Oracle yet — the output above is formatted for manual entry. Open
the account in Gainsight → Timeline (or Success Plans) to enter it."*

Don't apologize. Don't narrate the Phase 2 roadmap unprompted.

---

## Validated write recipe (Phase 2 — Bluhm B3)

When writes unlock, use the validated two-step chain — don't guess payloads:

1. **`prepare_*`** (e.g. `prepare_cta`) → returns the option set + required fields for this org.
2. Build the payload from what prepare returned, then **`create`**.
3. **Discover-then-pass for required custom fields:** the first write may fail with an error naming a
   missing required field + its allowed values. Read it, add the field, retry, and **cache the
   requirement for the session** so the next write doesn't repeat the round-trip.

This closes the `prepare_sp` gap from Sentinel v3.7 — the chain is validated, not assumed.

---

## Confirmation prompt (Phase 2)

Required before any write executes, even if all checks pass. No exceptions.

```
About to log to Gainsight Timeline:
  Account: [name]
  Type:    [activity type]
  Subject: [subject line]

Confirm? (yes / no)
```

No explicit confirmation → do not call the write tool. "Go ahead" / "yes" counts. Ambiguous → ask
once more, then stop.

---

## Bulk write hard block

Any write targeting more than one account in a single operation → refuse unconditionally:
*"Bulk writes are disabled. Run these one account at a time."*

No exceptions. Not in CS Ops run-as. Not in test mode.

---

## Phase 2 unlock path

To enable a write: add it to the allowed list, define its confirmation template, set scope, and
confirm the `prepare_*` → `create` recipe against a real org. Update this file on every change. State
plainly that the write runs as the connected seat. Bulk block applies to all writes unless explicitly
listed bulk-safe.

---

*2026.06.13 — Ported from Sentinel v3.7 write-gateway.md → Oracle. Added the "writes run as the connected seat / verify_downward is courtesy not control" statement (charter pre-Phase-2 requirement). Added Bluhm B3 validated prepare→create recipe + discover-then-pass custom-field recovery (closes the prepare_sp gap). Sentinel → Oracle rebrand. No customer data present (placeholders only).*
