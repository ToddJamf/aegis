# Radar — Scoring Reference
# Aegis stack 2026.06.13
# Bundled (local). Load when computing renewal (R1–R5) or expansion (E1–E4) scores in the render layer.

Full formulas with edge cases. All computation happens in the render layer — never in queries.

---

## Renewal Readiness Score (R1–R5, max 100)

Higher = healthier renewal. Lower = more urgent.

### R1 · Renewal urgency (max 25)

```js
var days = daysUntilRenewal(account.RenewalDate);  // negative if past due
var status = account.Renewal_Status__gc_PicklistLabel;

var R1;
if (['Escalated','Late Escalated','Notice of Churn'].includes(status)) {
    R1 = 0;
} else if (status === 'Late') {
    R1 = 5;
} else if (status === 'Churn') {
    R1 = 0;
} else if (days <= 0) {
    R1 = 2;   // renewal date passed, still Open
} else if (days <= 30) {
    R1 = Math.round(2 + (days / 30) * 8);   // 2-10
} else if (days <= 60) {
    R1 = Math.round(10 + ((days - 30) / 30) * 10);  // 10-20
} else {
    R1 = Math.round(20 + ((days - 60) / 120) * 5);  // 20-25
}
R1 = Math.min(25, Math.max(0, R1));
```

**Null RenewalDate:** Exclude account from the renewal lane entirely. Cannot triage without a date.

### R2 · Account health (max 25)

```js
var scoreMap = {A: 25, B: 20, C: 12, D: 5, F: 0};
var R2 = scoreMap[account.Overall_Score__gc_PicklistLabel] ?? 12;  // null → treat as C

if (account.Churn_Risk_Identified__gc)    R2 = Math.min(R2, 10);
if (account.Open_CSE_Risk_Mitigation__gc) R2 = Math.min(R2, 8);
```

**Null Overall_Score:** Default to 12 (C equivalent). Render score display as "—".

### R3 · Product adoption (max 25)

**Tech-touch detection:** If `account.CsmName === "Jamf Digital Team"` OR `account.CsmName` is null/empty, the account is tech-touch (no individual CSM). Use the tech-touch scoring path. Otherwise use the standard path.

**Tech-touch path (SMB / digital accounts):**
```js
var rel = perAccount[account.Gsid] || {};
var techTouch = !account.CsmName || account.CsmName === 'Jamf Digital Team';

if (techTouch) {
    // Primary product = product row with highest Paid_For_Quantity AND Active30 > 0
    // Ignore products with zero active devices — they may be unactivated bundle items
    var primaryRow = rel.rows
        ? rel.rows.filter(r => (r.Active30 || 0) > 0)
                  .sort((a,b) => (b.Paid||0) - (a.Paid||0))[0]
        : null;

    var primaryDeployed = primaryRow ? (primaryRow.Deployed || 0) : 0;
    var primaryActive   = primaryRow ? (primaryRow.Active30 || 0) : 0;
    var primaryPaid     = primaryRow ? (primaryRow.Paid || 1) : 1;
    var utilRatio       = primaryPaid > 0 ? primaryActive / primaryPaid : 0;

    // Login from best-available product row (any product with a login date)
    var loginStr  = rel.latest_login || null;
    var loginDays = loginStr ? daysSince(loginStr) : 999;

    var depScore    = (primaryDeployed / 100) * 12;           // 0-12
    var utilScore   = Math.min(8, utilRatio * 8);             // 0-8
    var loginScore  = Math.max(0, 5 - loginDays / 14);       // 0-5, zero at 70 days

    R3 = Math.round(Math.min(25, depScore + utilScore + loginScore));

} else {
    // Standard path (managed accounts with individual CSM)
    var deployed    = rel.worst_deployed_pct ?? 0;
    var loginDays   = rel.latest_login ? daysSince(rel.latest_login) : 999;
    var activeRatio = rel.total_paid > 0 ? rel.total_active_30d / rel.total_paid : 0;
    var depScore    = (deployed / 100) * 15;
    var loginScore  = Math.max(0, 7 - loginDays / 10);
    var activeScore = Math.min(3, activeRatio * 3);
    R3 = Math.round(Math.min(25, depScore + loginScore + activeScore));
}
```

**Bundle activation gap — flag, not a score penalty:**
After computing R3, scan for products where `Paid > 0` AND `Active30 === 0`. These represent purchased-but-unused products. Surface as a chip on the row: `"⚠️ Security Cloud not activated"` or `"⚠️ Protect not deployed"`. This is an expansion/activation conversation, not a churn signal — do not subtract from R3.

```js
var bundleGaps = rel.rows
    ? rel.rows.filter(r => (r.Paid||0) > 0 && (r.Active30||0) === 0 && r.Product !== 'Pro')
              .map(r => r.Product)
    : [];
// Render as chips on row: "⚠️ {product} not activated"
```

**No relationship rows:** R3 = 0. Render adoption fields as "—".

### R4 · Activity recency (max 10)

**Tech-touch accounts use last admin login, not last timeline entry.**
Timeline entries are CSM-generated. For tech-touch accounts, the CSM is "Jamf Digital Team" and entries are rare or absent — using them would score every SMB account at 0 regardless of actual product engagement.

```js
var techTouch = !account.CsmName || account.CsmName === 'Jamf Digital Team';
var rel = perAccount[account.Gsid] || {};

var R4, daysSinceLast, lastSignal;

if (techTouch) {
    // Use last admin login from relationship data — primary engagement proxy for SMB
    var loginStr = rel.latest_login || null;
    daysSinceLast = loginStr ? daysSince(loginStr) : 999;
    lastSignal = loginStr ? 'Last login ' + daysSinceLast + 'd ago' : 'No login on record';
    R4 = loginStr
        ? Math.round(Math.max(0, 10 - daysSinceLast / 9))  // zero at 90+ days
        : 0;
} else {
    // Standard path — use timeline entry
    var last = account.Last_Timeline_Entry_All__gc;
    daysSinceLast = last ? daysSince(last) : 0;
    lastSignal = last ? 'Last contact ' + daysSinceLast + 'd ago' : 'No activity on record';
    R4 = last
        ? Math.round(Math.max(0, 10 - daysSinceLast / 9))
        : 0;
}
```

**Display label change for tech-touch:** The "Last contact" dimension label in the dropdown becomes **"Last admin login"** for tech-touch accounts. Same position in the score breakdown, different label and data source.

### R5 · Expansion signal (max 15)

```js
var expLevel  = account.Expansion_Rediness_Level__gc ?? 0;   // note: "Rediness" typo in field name
var staircase = account.Staircase_Overall_Score__gc ?? 0;

var R5 = Math.round(Math.min(15, (expLevel * 0.1) + (staircase * 0.05)));
```

### Total & verdict

```js
var total = R1 + R2 + R3 + R4 + R5;  // 0-100

var verdict;
if (account.Auto_Renewal__gc) {
    // Downgrade one tier
    if      (total < 50) verdict = 'watch';
    else                 verdict = 'on-track';
} else {
    if      (total < 50) verdict = 'act-now';
    else if (total < 75) verdict = 'watch';
    else                 verdict = 'on-track';
}
```

### Plain-language reason

Pick the lowest-scoring dimension relative to its max. Return a human-readable sentence.

```js
// Display labels — never show R1/R2/R3 codes to users
// R4 label changes based on tech-touch detection
var dims = [
    { label: 'Time to renewal',    score: R1, max: 25, reason: renewalReason(days, status) },
    { label: 'Health score',       score: R2, max: 25, reason: healthReason(account) },
    { label: 'Deployment & usage', score: R3, max: 25, reason: adoptionReason(rel, techTouch, primaryDeployed, loginDays) },
    { label: techTouch ? 'Last admin login' : 'Last contact',
                                   score: R4, max: 10, reason: lastSignal },
    { label: 'Growth signal',      score: R5, max: 15, reason: expansionReason(expLevel) },
];
var worst = dims.reduce((a, b) => (a.score / a.max) <= (b.score / b.max) ? a : b);
return worst.reason;

// Reason generators:
function renewalReason(days, status) {
    if (status !== 'Open') return 'Renewal status: ' + status;
    if (days <= 0) return 'Renewal date passed';
    return 'Renewal in ' + days + ' days';
}
function healthReason(a) {
    var base = 'Health ' + (a.Overall_Score__gc_PicklistLabel || '—');
    if (a.Churn_Risk_Identified__gc) return base + ' · churn risk flagged';
    return base;
}
function adoptionReason(rel, deployed, loginDays) {
    if (!rel.latest_login) return 'No admin login on record';
    if (loginDays > 60)    return 'No login in ' + loginDays + ' days';
    if (deployed < 25)     return Math.round(deployed) + '% deployed on ' + rel.weakest_product;
    return Math.round(deployed) + '% deployed · login ' + loginDays + 'd ago';
}
function activityReason(last, days) {
    if (!last) return 'No activity on record';
    return 'Last touch ' + days + 'd ago';
}
function expansionReason(level) {
    if (!level) return 'No expansion signal';
    return 'Expansion readiness: ' + level + ' / 100';
}
```

---

## Expansion Score (E1–E4, max 100)

Higher = stronger opportunity.

### E1 · Expansion readiness (max 35)

```js
var E1 = Math.round(((account.Expansion_Rediness_Level__gc ?? 0) / 100) * 35);
```

### E2 · Over-utilization (max 35)

```js
var rel = perAccount[account.Gsid] || {};
var E2;
if (rel.over_utilized && rel.total_paid > 0) {
    // Guard: total_paid = 0 would produce Infinity — skip overage calc
    var overagePct = (rel.total_active_30d - rel.total_paid) / rel.total_paid;
    E2 = Math.round(Math.min(35, 25 + overagePct * 50));
} else if (rel.over_utilized && rel.total_paid === 0) {
    // Active devices exist but paid quantity is zero — data anomaly, score conservatively
    E2 = 25;
} else if (rel.enrolled_over_licensed) {
    E2 = 20;
} else {
    var utilRatio = rel.total_paid > 0 ? rel.total_active_30d / rel.total_paid : 0;
    E2 = Math.round(Math.min(15, utilRatio * 15));
}
```

### E3 · Account health (max 20)

```js
var healthMap = {A: 20, B: 16, C: 10, D: 4, F: 0};
var E3 = healthMap[account.Overall_Score__gc_PicklistLabel] ?? 10;
```

```js
// Expansion tab display labels: 'Expansion readiness' · 'Device utilization' · 'Account health' · 'Staircase signal'
// Never show E1/E2/E3/E4 codes in UI
```

### E4 · Staircase expansion (max 10) — conditional

```js
var E4 = 0;
if (staircaseConnected) {
    E4 = Math.round(((account.Staircase_Overall_Score__gc ?? 0) / 100) * 10);
}
// E4 = 0 when not connected — dimension row hidden in artifact, not rendered as N/A
```

### Total & verdict

```js
var total = E1 + E2 + E3 + E4;  // E4 is 0 when Staircase not connected, never NaN

var verdict;
if      (rel.over_utilized) verdict = 'expand-strong';  // always surface
else if (total >= 75)       verdict = 'expand-strong';
else if (total >= 50)       verdict = 'expand-mid';
else                        verdict = 'expand-low';
```

**Sort order:** Over-utilized accounts always first (regardless of score), then by total descending.

---

## Null / missing data rules

| Field | Null treatment |
|---|---|
| `RenewalDate` | Exclude from renewal lane entirely |
| `Overall_Score__gc` | Default C (score 12/20 for R2, 10/20 for E3), display "—" |
| `Arr` | Excluded by query filter ($10K floor) — won't appear |
| `Last_Timeline_Entry_All__gc` | Standard accounts: R4 = 0, reason = "No activity on record". Tech-touch: ignored — use login from relationship instead |
| `Deployed__gc` on relationship row | Skip row in aggregation |
| `Expansion_Rediness_Level__gc` | E1 = 0 |
| `Staircase_*` fields | Default 0, score as if absent |

---

## ARR display in scored views

Scoring never reads ARR for math — the $10K floor is a query filter, not a score input. When a scored row *displays* ARR (renewal lane, expansion lane, list caps), apply `references/arr-policy.md`: full `$` figure inside the requester's owned scope, `ARR_Band__gc` label for accounts shown in out-of-scope list caps.

---

*2026.06.13 — Ported from radar-2026-06-03 archive (was scoring.md "Renewal View" v1.2). Retitled/rebranded Radar, CalVer header. Scoring logic ported faithfully and unweakened — R1–R5, E1–E4, tech-touch path, bundle-activation flag, E2 div/zero guard, null rules all intact. `Expansion_Rediness_Level__gc` field-name typo ("Rediness") preserved — it is the exact Gainsight API name. Added ARR-display cross-ref to arr-policy.md. No customer/employee PII in source — confirmed clean.*
