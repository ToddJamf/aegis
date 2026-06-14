# Oracle — Terminology, CS (full detail)
# Aegis stack 2026.06.13
# Bundled (local). Load for explicit glossary lookups (CSM/Leader) and when full cadence, product, or field detail is needed.

For query routing and phrasing translation only — load `synonym-map.md` instead (lightweight, ~1k tokens).
Not for Not-CS users.

Updated 2026-04-29 from probes + KPI doc + Aegis deck.

---

## Contents
- Jamf CS field values (Customer Category, Sentiment, Renewal Status, ARR Band, etc.)
- Territory fields
- Gainsight terms (full)
- Staircase terms
- Confidentiality rules

*(Common phrasings → field mapping lives in `synonym-map.md` — load that for routing, not this file.)*

---

## Jamf CS field values

**Customer Category** (`Customer_Category__gc` PICKLIST) — KPI dimension.

| Value | Meaning | Engagement cadence |
|-------|---------|-------------------|
| Spotlight | Top-priority strategic | Bi-weekly |
| Retain | Established, maintain | Every 3 weeks |
| Grow | Expansion target | Monthly |
| React | Lower-touch / reactive | CTA-driven, ~6 months |

Universal tile order: **Spotlight → Retain → Grow → React**.

**Tier** (`Tier__gc`) — superset of Customer Category, adds `Global Accounts - Embedded` and `Top Customer`. Use Customer_Category for KPI; Tier only when user names it explicitly.

**CSM Sentiment** (`CSM_Sentiment__gc`) — manual rating. A=Excellent · B=Good · C=Neutral · D=At risk · F=Critical. Stale threshold: 90 days (`CSM_Sentiment_Last_Updated__gc`).

**Renewal Status** (`Renewal_Status__gc`) — Open · Late · Escalated · Late Escalated · Notice of Churn · Churn.

The formal escalation set is `[Escalated, Late Escalated]` only. `Notice of Churn` is post-escalation (different play). `Churn` = lost. (Escalation triage and team risk views live in Radar — Oracle reads these values for brief context only.)

**CSM Priority** (`CSM_Priority__gc`) — High Touch · Medium Touch · Low Touch.

**ARR Band** (`ARR_Band__gc`) — `<$10k | $10-50k | $50-100k | $100-250k | $250k+`. Confidential.

**Engagement Model** (`CS_Segment__gc`) — Enterprise · Enterprise-Embedded · Enterprise-Named · Mid-Market · Small Market. Override field: `CS_Segment_Override__gc`.

---

## Territory fields

- `CS_Territory_Region__gc` — AMER / EMEIA / APAC / LATAM
- `CS_Territory_Segment__gc` — Named / Strategic / Upmarket / Commercial / Education / SMB
- `CS_Territory_Team__gc` — full label (e.g., `AMER - Named`)

Used for territory-aware scoping in Brief. CSMs are scoped to own book by default; same-region cross-book is allowed without prompt; cross-region triggers a soft flag. Leaders have no region gate.

---

## Employee type

`Employee_Type__gc` on gsuser — Individual Contributor / Leadership / Executive.

Used as a tier hint only. Tier resolution is evidence-based:
- CSM tier = has CS role (`Csm`, `Account_Owner__gc`, etc.) on ≥1 active company AND no CS-org reports
- Leader tier = manages ≥1 active user in the CS org (any level, recursive)
- Not CS = neither

---

## Pre-computed engagement fields (read, never compute)

- `Engaged_This_Period_by_CSM__gc` (BOOLEAN) — true if within cadence window per Customer Category
- `Last_Timeline_Entry_Engagement__gc` (DATE) — date of last logged engagement activity

---

## Jamf Products / Relationships

Gainsight `relationship` records represent individual Jamf product instances under a company. The **Family** field maps to the product name.

### Relationship types (product families)

| Family | Product | Notes |
|--------|---------|-------|
| Pro | Jamf Pro | MDM core. Most accounts have 2 envs: Production + Sandbox. |
| Protect | Jamf Protect | Endpoint security. Often licensed but under-deployed — check enrollment %. |
| Connect | Jamf Connect | Identity/SSO. May appear as a named relationship ("Connect") not a CEN-. |
| Security Cloud | Jamf Security Cloud | Network/threat. Less common. |

### Environment naming

- **CEN-XXXXXXX** — Cloud Environment Name. Jamf-hosted tenant ID. May also appear as product name ("Connect") for older records.
- **Environment Type** — Standard (production) or Sandbox (dev/test, typically 10 free licenses per product).
- **Last Login** — last user activity in that environment. Stale > 60d is a risk signal.

### Key enrollment fields (queryable directly on `relationship` object — verified 2026-05-04)

| Field | Meaning |
|-------|---------|
| Paid For Quantity | Licensed seats |
| Total Enrolled | Devices actively enrolled |
| Deployed % | Total Enrolled / Paid For Quantity |
| Device Based Qty | macOS/iOS/tvOS device licenses |
| User Based Qty | User-seat licenses (rare) |

These fields are available on the `relationship` object via standard `run_query`. See `field-registry.md` for the verified full select payload.

### Adoption signals to surface in briefs

- **Deployed % < 50%** — adoption gap. Flag in Products section.
- **Deployed % < 10%** — critical under-deployment. Escalate in Next Focus.
- **Last Login stale (>60d)** — product may be inactive. Surface as churn signal.
- **Protect licensed but <10% deployed** — common pattern. IT Security team often owns Protect separately from Mac admin. Named stakeholder for Protect is typically different from Jamf Pro contact.

### Query notes

- Safe to select on `relationship`: `Name`, `Gsid`, plus all enrollment fields above. Avoid `StatusId__gr.Name` and `RelationshipTypeId__gr.Name` — both are PICKLIST fields that throw P_5068.
- `Cloud_Family__gc` and `Environment_Type__gc` are PICKLIST fields — use direct select (auto-returns `_PicklistLabel`). Do not use `__gr.Name`.

---

## Gainsight terms

- **CTA** — Call To Action. Object: `call_to_action`. `IsClosed` flag (boolean — always pass `false`, not the string "false"). May have sub-tasks (`cs_task`).
- **Success Plan** — multi-step strategic plan. Object: `cta_group` (misleading API name). Has `IsClosed`, `PercentComplete`.
- **Objective** — line item within an SP. Treated as special CTA in queries.
- **Scorecard** — health system. Score 0–100, label A–F. Tool: `ask_scorecard`. Common: `Enterprise - Weighted` and `Enterprise - Unweighted`.
- **Measure** — scorecard component. `measures_flat` often empty in tenant.
- **Timeline** — activity log. Object: `activity_timeline`. Mix of human-logged (Meeting/Email) and system-generated (Milestone/Stage Change). Filter field: `GsCompanyId` (not `CompanyId` — P_5005 if wrong).
- **Milestone** — system-generated timeline event. Author: "System Administrator."
- **Company vs Relationship** — Company = top-level account. Relationship = sub-product (see Jamf Products section above). Default Company unless user says "relationship."
- **GSID** — 36-char ID. Cache and reuse.

**Field naming:**

| Suffix | Meaning |
|--------|---------|
| `__gc` | Custom field (Gainsight Custom) |
| `__gr` | Lookup target — `Field__gr.Name` resolves LOOKUP fields |
| `_PicklistLabel` | Auto-companion returning human label for PICKLIST GSID |

LOOKUP fields support `__gr.Name`. PICKLIST fields don't (P_5068 error if attempted).

---

## Staircase terms

- **Signal** — risk/opportunity indicator from communications. Retrieve via `staircase_analyze_account` (account-scoped, preferred) or `staircase_query` (global search only — not for capability use). Returns narrative + risk level.
- **Evidence** — source comms (emails, meetings) supporting a signal. `evidences[]` with ID, title, text, URL.
- **"No reply found"** — no signal exists. NOT an error. Render as "no recent Staircase signal."

---

## Confidentiality

Sensitive — never log to memory or surface in external content:
- `ARR_Band__gc` and any renewal $$$
- Customer name + ARR pairing
- Pre-public roadmap mentioned in CTAs/timeline
- Individual CSM performance comparisons

ARR $ renders only to account's CSM and their recursive manager chain. Anyone outside → `ARR_Band__gc` only, labeled `[restricted]`. See ARR confidentiality rules in `arr-policy.md`.

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
