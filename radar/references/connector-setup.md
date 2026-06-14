# Radar — Connector setup
# Aegis stack 2026.06.13
# Bundled (local). Load when Step 0 detects a missing connector, or when a user hits permission prompts on Gainsight or Staircase tools.

---

## Not installed

### Gainsight not live (blocking)

Render this prompt and stop. Do not proceed with any data capability.

> ⚙️ **Gainsight isn't connected yet.**
> To use Radar, you'll need to add the Gainsight connector in Cowork:
> 1. Open **Cowork → Settings → Connectors**
> 2. Find **Gainsight** and click **Add** (or **Always allow** if it's listed but not enabled)
> 3. Come back and try again — Radar will pick it up automatically.
>
> Radar needs Gainsight live for every view — renewal, expansion, plate, compliance, heat-map.

If Staircase is also missing, append a secondary note below — do not give it equal weight:

> *Staircase AI also isn't connected. Once Gainsight is set up, add Staircase the same way for full risk and expansion signals.*

### Staircase not live (degraded — session-once)

Proceed in degraded mode. Append this note once to the first degraded output only — suppress for the remainder of the session:

> *⚠ Staircase isn't connected — risk signals are unavailable, and the expansion Staircase dimension (E4) is omitted rather than scored. To add it: Cowork → Settings → Connectors → Staircase AI → Always allow.*

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

Gainsight: `resolve_user`, `resolve_customer`, `run_query`, `fetch_cta_list`, `fetch_success_plan_list`, `fetch_timeline_activity_list`, `ask_scorecard`, `get_object_metadata`, `get_picklist_values`, `get_records`

Staircase: `staircase_account_lookup`, `staircase_analyze_account`, `staircase_fetch_evidence`, `staircase_query` (global search only — not for capability use; see field-registry.md)

After one-time allow per tool, prompts stop for the session.

### Connector not installed

If Gainsight or Staircase doesn't appear in Cowork → Settings → Connectors at all, the connector needs to be added. Point the user to their Cowork admin or CS Ops.

This is a Cowork client behavior — the skill can't bypass it.

---

*2026.06.13 — Created for Radar, structure based on oracle/references/connector-setup.md. Retitled Radar; added resolve_customer to the Gainsight per-tool list and noted the E4 (Staircase) dimension omission in degraded mode. Generic — no customer data.*
