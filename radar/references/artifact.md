# Radar — Artifact Reference
# Aegis stack 2026.06.13
# Bundled (local). Load when building the HTML artifact (render step).

Full HTML template spec for Radar's renewal + expansion view.

---

## Structure

```
artifact
├── header (title + subtitle)
├── tabs (Renewal triage | Expansion pipeline)
├── tab-panel: renewal
│   ├── stats-bar (accounts · ARR at stake · act now · watch · on track)
│   ├── lane: act-fast (0-30 days)
│   │   ├── account-row (collapsed)
│   │   └── account-row (expanded — shows dropdown)
│   ├── lane: coming-up (31-60 days)
│   └── lane: on-horizon (61-180 days)
└── tab-panel: expansion
    ├── stats-bar (accounts · over-utilized · strong · potential)
    ├── lane: over-utilized (floated to top)
    └── lane: strong-signal (score 75+)
```

---

## Jamf brand tokens

```css
:root {
  --jamf-blue:    #056AE6;
  --jamf-haiti:   #100F2F;
  --jamf-pearl:   #F9F7F1;
  --jamf-deepsea: #00348A;
}
```

Font: `'Inter', 'Helvetica Neue', Helvetica, Arial, sans-serif`
Load: `<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap" rel="stylesheet">`

---

## Skeleton loading state

Render immediately on artifact open, before queries return. Replace with real rows on data load.

```html
<div class="acct-row skeleton">
  <div class="acct-main" style="pointer-events:none">
    <div class="skel-block" style="width:140px;height:14px;border-radius:4px"></div>
    <div class="skel-block" style="width:60px;height:14px;border-radius:4px;margin-left:12px"></div>
    <div class="skel-block" style="width:48px;height:14px;border-radius:4px;margin-left:12px"></div>
    <div class="skel-block" style="width:120px;height:22px;border-radius:99px;margin-left:auto"></div>
  </div>
</div>

<style>
.skel-block {
  background: var(--color-border-tertiary);
  animation: skel-pulse 1.4s ease-in-out infinite;
  display: inline-block;
}
@keyframes skel-pulse {
  0%,100% { opacity:1 }
  50%      { opacity:0.35 }
}
</style>
```

Render 3 skeleton rows per lane (9 total across 3 lanes). Stats-bar also shows skeleton values ("—") until data loads.

---

## Account row — collapsed

```html
<div class="acct-row" id="row-{gsid}">
  <button class="acct-main"
          onclick="toggle('{gsid}')"
          aria-expanded="false"
          aria-controls="drop-{gsid}">
    <span class="acct-name">{account.Name}</span>
    <span class="acct-days">
      <i class="ti ti-calendar" aria-hidden="true"></i> {daysLabel}
    </span>
    <span class="acct-arr">{arrDisplay}</span>
    <!-- Auto-renewal chip — render on BOTH tabs when Auto_Renewal__gc = true -->
    {autoRenewalChip}
    <span class="verdict {verdictClass}">{score} / 100 · {verdictLabel}</span>
    <span class="reason">{reasonText}</span>
    <i class="ti ti-chevron-down chevron" id="chev-{gsid}" aria-hidden="true"></i>
  </button>
  <div class="acct-drop" id="drop-{gsid}" role="region" aria-label="{account.Name} details">
    <!-- dropdown content — see below -->
  </div>
</div>
```

**Auto-renewal chip:**
```html
<!-- Only render when Auto_Renewal__gc = true -->
<span class="auto-chip">
  <i class="ti ti-trending-up" aria-hidden="true"></i> Auto ↑
</span>
```

**`daysLabel` logic:**
```js
if (days <= 0)       return 'Today';
if (days === 1)      return 'Tomorrow';
if (days <= 30)      return days + ' days';
return renewalDate.toLocaleDateString('en-US', {month:'short', day:'numeric'});
```

**`arrDisplay` logic — applies `references/arr-policy.md`:**
```js
// In-scope (own book / leader scope / CS Ops run-as): full $ figure.
// Out-of-scope rows surfaced in list caps: ARR_Band__gc label, not the exact figure.
if (!inScope(account)) return account.ARR_Band__gc_PicklistLabel || '';
var arr = account.Available_to_Renew__gc || account.Arr;
if (!arr) return account.ARR_Band__gc_PicklistLabel || '';
return '$' + (arr >= 1000 ? (arr/1000).toFixed(1) + 'K' : arr);
```

---

## Account row — expanded dropdown

Three sections. Render in order.

### Section 1 — Score breakdown (always)

```html
<div class="acct-drop" id="drop-{gsid}">
  <div class="drop-section-title">
    {tab === 'renewal' ? 'Renewal readiness score' : 'Expansion score'}
  </div>
  <div class="score-summary">
    <span class="score-num">{total}</span>
    <span class="score-denom">/ 100</span>
    <span class="verdict {verdictClass}" style="margin-left:auto">{verdictLabel}</span>
  </div>

  <!-- Dimension rows — use plain English display labels only, never R1/R2/E1/E2 codes -->
  <!-- Renewal tab labels: Time to renewal · Health score · Deployment & usage · Last contact · Growth signal -->
  <!-- Expansion tab labels: Expansion readiness · Device utilization · Account health · Staircase signal -->
  <!-- Omit Staircase signal row entirely if Staircase MCP not connected -->
  <div class="dim-row">
    <span class="dim-label">{displayLabel}</span>
    <div class="dim-track">
      <div class="dim-fill {fillClass}" data-width="{pct}%" style="width:0%"></div>
    </div>
    <span class="dim-score">{score} / {max}</span>
    <span class="dim-note">{note}</span>
  </div>
  <!-- repeat per dimension -->

  <!-- Example render (renewal tab):
    Time to renewal     ████░░░░░░░░  9 / 25  · 26 days · Open
    Health score        █████░░░░░░░  12 / 25  · C
    Deployment & usage  ░░░░░░░░░░░░  — / 25  · Loading...
    Last contact        ████████░░░░  7 / 10   · May 6 (27 days ago)
    Growth signal       ░░░░░░░░░░░░  0 / 15  · No signal detected
  -->
```

**`fillClass` logic:**
```js
var pct = score / max;
if (pct < 0.3)      return 'fill-low';    // red
else if (pct < 0.6) return 'fill-mid';    // amber
else if (pct < 0.8) return 'fill-hi';     // blue
else                return 'fill-great';  // green
```

### Section 2 — Staircase (conditional)

Three states — see SKILL.md "Staircase section — failed lookup":

```html
<!-- State A: MCP connected, lookup succeeded -->
<div class="drop-sub">
  <span class="badge badge-staircase">
    <i class="ti ti-brand-snowflake" aria-hidden="true"></i> Staircase
  </span>
  <div class="assess-text">{staircase_result.answer}</div>
</div>

<!-- State B: MCP connected, lookup failed -->
<div class="drop-sub">
  <span class="badge badge-staircase">
    <i class="ti ti-brand-snowflake" aria-hidden="true"></i> Staircase
  </span>
  <div class="assess-text" style="color:var(--color-text-tertiary)">
    No Staircase match found for {account.Name}. The account may not be synced yet.
  </div>
</div>

<!-- State C: MCP not connected — omit entire section, no header, no message -->
```

### Section 3 — Claude synthesis (always)

```html
<div class="drop-sub">
  <span class="badge badge-claude">
    <i class="ti ti-sparkles" aria-hidden="true"></i> Claude assessment
  </span>
  <div class="assess-text">{claudeSynthesis}</div>
</div>
```

**Claude synthesis template:**
```
{Account name} {context}.
{One signal sentence from worst dimension}.
{One action sentence}.
```

Examples (account names genericized):
- *"Acme Co renews today with a D health score and no CSM on record. Pro deployment is solid but Connect and Security Cloud show zero active devices. Call today — lead with deployment data, find out if there's been an admin change."*
- *"Acme Group is running 27% over their licensed device count. Health is A, auto-renewing, and renewal is 112 days out. Warmest expansion call in your book — they need the licenses and the relationship is solid."*

---

## Empty states

```html
<!-- Empty swim lane -->
<div class="empty-state">
  <i class="ti ti-check" aria-hidden="true"></i>
  No renewals in this window
</div>

<!-- No accounts after $10K filter -->
<div class="empty-state">
  No accounts in your book match the $10K ARR floor.
  This may indicate ARR data hasn't synced — check with CS Ops.
</div>

<!-- Expansion tab: no signal -->
<div class="empty-state">
  No strong expansion signals detected in your book right now.
</div>
```

---

## Heads up strip (renewal tab only — renders above swim lanes)

Only rendered when accounts qualify (see SKILL.md Step 4b). If zero qualifying accounts, omit entirely — no empty state.

```html
<div class="heads-up">
  <div class="heads-up-title">
    <i class="ti ti-alert-triangle" aria-hidden="true"></i>
    Heads up · {N} account{s} outside your renewal window need attention
  </div>
  <div class="heads-up-rows">
    <!-- One compact row per account, max 5 -->
    <div class="hu-row">
      <span class="hu-name">{account.Name}</span>
      <span class="hu-renewal">Renews {dateLabel}</span>
      <span class="hu-arr">{arrDisplay}</span>
      <span class="hu-flag {flagClass}">{flagLabel}</span>
      <span class="hu-reason">{flagReason}</span>
    </div>
    <!-- If > 5 accounts -->
    <div class="hu-more">
      <button onclick="sendPrompt('show all flagged accounts outside my renewal window')">and {N} more ↗</button>
    </div>
  </div>
</div>
```

**Flag logic:**

| Condition | `flagClass` | `flagLabel` |
|---|---|---|
| Status: Escalated / Late Escalated | `flag-escalated` | 🚨 Escalated |
| Status: Notice of Churn | `flag-churn` | 🚨 Notice of churn |
| `Churn_Risk_Identified = true` | `flag-risk` | ⚠️ Churn risk flagged |
| `Churn_Risk_Level >= 3` only | `flag-risk` | ⚠️ Churn risk level {N} |

**CSS:**
```css
.heads-up { border:0.5px solid var(--color-border-warning); border-radius:var(--border-radius-lg); background:var(--color-background-warning); padding:10px 14px; margin-bottom:16px; }
.heads-up-title { font-size:12px; font-weight:500; color:var(--color-text-warning); margin-bottom:8px; display:flex; align-items:center; gap:6px; }
.hu-row { display:flex; align-items:center; gap:10px; padding:5px 0; border-top:0.5px solid var(--color-border-warning); font-size:13px; }
.hu-name { font-weight:500; color:var(--color-text-primary); flex:1; min-width:0; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
.hu-renewal,.hu-arr,.hu-reason { font-size:12px; color:var(--color-text-secondary); white-space:nowrap; }
.hu-reason { font-style:italic; }
.hu-flag { font-size:11px; font-weight:500; padding:2px 8px; border-radius:99px; white-space:nowrap; }
.flag-escalated,.flag-churn { background:var(--color-background-danger); color:var(--color-text-danger); }
.flag-risk { background:var(--color-background-warning); color:var(--color-text-warning); border:0.5px solid var(--color-border-warning); }
.hu-more button { font-size:12px; color:var(--color-text-info); background:none; border:none; cursor:pointer; padding:4px 0; }
```

---

## Toggle function (canonical — use exactly this)

```js
function toggle(id) {
  var drop = document.getElementById('drop-' + id);
  var btn  = drop ? drop.previousElementSibling : null;
  var chev = document.getElementById('chev-' + id);
  if (!drop) return;

  var isOpen = drop.classList.toggle('open');
  if (chev) chev.classList.toggle('open', isOpen);
  if (btn)  btn.setAttribute('aria-expanded', String(isOpen));

  if (isOpen) {
    drop.querySelectorAll('.dim-fill').forEach(function(el) {
      var target = el.dataset.width || '0%';
      el.style.width = '0%';
      requestAnimationFrame(function() {
        requestAnimationFrame(function() { el.style.width = target; });
      });
    });
  }
}
```

---

## Verdict class map

| Verdict | CSS class | Label |
|---|---|---|
| Act now | `act-now` | 🚨 Act now |
| Watch | `watch` | ⚠️ Watch |
| On track | `on-track` | ✅ On track |
| Expansion strong | `expand-strong` | 🚀 Strong |
| Expansion mid | `expand-mid` | 📈 Potential |
| Expansion low | `expand-low` | Low signal (render muted, no icon, smaller font) |

---

*2026.06.13 — Ported from radar-2026-06-03 archive (was artifact.md "Renewal View" v1.1). Retitled/rebranded Radar, CalVer header. Render spec ported faithfully — skeleton loading, aria-expanded, auto-renewal chip both tabs, score-bar animation, three-state Staircase section, empty states, heads-up strip, canonical toggle. Two changes vs. source: arrDisplay now defers to references/arr-policy.md (in-scope $ vs. out-of-scope band); three customer-name examples genericized to Acme Co / Acme Group. No other PII — confirmed clean.*
