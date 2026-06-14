# Oracle — Connector setup
# Aegis stack 2026.06.13
# Bundled (local). Load when Step 0 detects a missing connector, or when a user hits permission prompts on Gainsight or Staircase tools.

---

## Not installed

### Gainsight not live (blocking)

Render this prompt and stop. Do not proceed with any data capability.

> ⚙️ **Gainsight isn't connected yet.**
> To use Oracle, you'll need to add the Gainsight connector in Cowork:
> 1. Open **Cowork → Settings → Connectors**
> 2. Find **Gainsight** and click **Add** (or **Always allow** if it's listed but not enabled)
> 3. Come back and try again — Oracle will pick it up automatically.
>
> In the meantime, glossary lookups still work. Try *"what is a CTA?"*

If Staircase is also missing, append a secondary note below — do not give it equal weight:

> *Staircase AI also isn't connected. Once Gainsight is set up, add Staircase the same way for full risk signals.*

### Staircase not live (degraded — session-once)

Proceed in degraded mode. Append this note once to the first degraded output only — suppress for the remainder of the session:

> *⚠ Staircase isn't connected — risk signals are unavailable. To add it: Cowork → Settings → Connectors → Staircase AI → Always allow.*

Do not repeat this nudge on subsequent outputs in the same session.

---

## Permission prompts

### Persistent allow (recommended)

If the user sees a permission prompt on every Gainsight or Staircase tool call, their Cowork client is set to per-prompt approval.

1. Open **Cowork → Settings → Connectors / MCP**
2. Find **Gainsight** and **Staircase AI**
3. Set each to **"Always allow"**

### Per-tool allow (fallback)

Some Cowork builds prompt per-tool inside the MCP rather than per-MCP. If prompts continue after the above, click "Always allow" the first time each tool fires:

Gainsight: `resolve_user`, `run_query`, `fetch_cta_list`, `fetch_success_plan_list`, `fetch_timeline_activity_list`, `ask_scorecard`, `get_object_metadata`, `get_picklist_values`, `get_records`

Staircase: `staircase_account_lookup`, `staircase_analyze_account`, `staircase_fetch_evidence`, `staircase_query` (global search only — not for capability use; see field-registry.md Pattern A / Pattern B)

After one-time allow per tool, prompts stop for the session.

### Connector not installed

If Gainsight or Staircase doesn't appear in Cowork → Settings → Connectors at all, the connector needs to be added. Point the user to their Cowork admin or CS Ops.

This is a Cowork client behavior — the skill can't bypass it.

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
