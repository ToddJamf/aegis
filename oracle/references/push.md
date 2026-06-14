# Oracle — Push capability
# Aegis stack 2026.06.13
# Bundled (local). Load when `oracle push` or "push to Slack" is triggered.

Pushes a brief from the current session to Slack.

---

## Trigger

`oracle push` / "push to Slack" / "send this to Slack" — only valid when a brief was delivered in the current session.

**Permission:** CSM, FLL. Not-CS → refused.

---

## Session scope check

If no brief has run in the current session → *"Nothing to push — run a brief first."* Do not proceed.

---

## Destination

- **Default:** requester's Slack DM (resolved from session identity email)
- **Override:** user specifies a channel — *"push to [#channel-name]"* → use that channel

---

## Phase 1 behavior (current)

Write gateway blocks — `slack_send_message` is not in the allowed writes list.

Response:
*"Slack push is coming in Phase 2 — copy from chat for now."*

No confirmation prompt. No partial execution.

---

## Phase 2 behavior (when unlocked)

1. Route through write gateway — confirmation prompt:
   ```
   Sending brief to [destination]:
     Account: [name]
     To: [DM or #channel]

   Confirm? (yes / no)
   ```
2. On confirm → `slack_send_message` to destination
3. Output: *"Sent."* — nothing else

**Phase 2 unlock:** add `slack_send_message` (brief push, single account, requester-scoped) to the allowed writes list in `write-gateway.md`.

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
