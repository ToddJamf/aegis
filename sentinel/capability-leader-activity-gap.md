# Sentinel — Capability 6: Activity Gap Report (Leader)
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-leader-activity-gap.md

Load on demand when a Leader-tier user asks about team activity gaps, compliance, or engagement.

**Triggers:** "team activity gap" / "team compliance" / "team activity report" / "where's my team struggling" / "AMER activity gaps" / "[region] compliance".

**Permission:** Leaders have full access — own team by default, region or global on request. No gate.

---

## Scope resolution

| User says | Scope |
|-----------|-------|
| "My team" / default | `Csm IN [leader.teamGsids]` |
| "[region]" (AMER / EMEIA / APAC / LATAM) | `CS_Territory_Region__gc = <region>` |
| "All" / "global" / "across the org" | No Csm filter (full tenant, honor ARR render rules) |

Cache resolved scope. Use as filter anchor for all calls.

---

## Sequence (~3-4 calls)

1. **CSM list resolution** — from `leader.teamGsids` (cached in Step 1) or re-query for region/global scope.

2. **Compliance roll-up** — single `run_query` on company, filtered to scope + grouped by Csm + `Engaged_This_Period_by_CSM__gc`. See `field-registry.md` "Compliance roll-up (Cap 6)".

   Per-CSM compliance % = engaged_count / total_count. Flag outliers >1σ below team avg as 🔴. Flag any 0% as 🚨.

3. **Stale account list** ($50k+, last engagement 60-90d or >90d) — single `run_query` on company within scope. See `field-registry.md` "Stale account list (Cap 6)".

4. **Top trends from timeline** — semantic RAG:

   ```python
   fetch_timeline_activity_list(
       where={"conditions": [
           {"fieldName": "GsCreatedByUserId", "operator": "IN",
            "value": [<scope_csm_gsids>], "alias": "A"},
           {"fieldName": "ActivityDate", "operator": "GTE",
            "value": ["<60d_ago>T00:00:00Z"], "alias": "B"}
       ], "expression": "A AND B"},
       contextual_user_query="recurring themes, blockers, customer initiatives, deployment topics, expansion conversations, escalation drivers, product feedback, and renewal discussions across the team's timeline notes",
       select=["Subject", "ActivityDate", "TypeName",
               "Ant__Sentiment__c",
               "GsCompanyId__gr.Name AS CompanyName",
               "GsCreatedByUserId__gr.Name AS AuthorName",
               "NotesPlainText AS Notes"],
       limit=40
   )
   ```

   Extract top 3 themes via keyword/topic clustering on Subject + Notes.

---

## Render

```
🎯 Sentinel — Activity Gap Report
[Leader name]  ·  [scope: team / AMER / global]  ·  [N] CSMs  ·  [M] accounts
[date]
[⚠ Staircase not connected — risk signals unavailable]  ← only if degraded

📊 COMPLIANCE ROLL-UP
   CSM              Book  Engaged  Compl%   vs Avg   Flag
   [csm 1]          ...   ...      ...%     +/-...   🟢/🟡/🔴/🚨
   ...
   ─────────────────────────────────────────────────────
   Scope total      ...   ...      ...%
   Scope avg (CSM rates): [%]

⚠️ STALE AT $50K+ — TOP 5 BY ARR
   [account]   $[arr or band]   [tier]   [csm]   Last engaged [date]
   ...

🧭 TOP 3 TRENDS — what your team is writing about
   (last 60 days · [N] timeline entries surfaced)
   1. [THEME] — [count] entries · Accounts: ... · Pattern: ...
   2. ...
   3. ...

🚩 DATA HYGIENE FLAGS  (if any)
   • [stale-engagement-field accounts]
   • [0% compliance CSMs — investigate, may indicate data hygiene]
   • [past-due renewal dates >180d]
```

ARR render: apply standard ARR confidentiality check per account row. Leaders outside the account.Csm manager chain see ARR Band only.

Output medium: HTML artifact (≥5-row roll-up triggers HTML).
