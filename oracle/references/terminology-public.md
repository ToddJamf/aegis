# Oracle — Terminology, Public (concept definitions)
# Aegis stack 2026.06.13
# Bundled (local). Load for Glossary lookups when the user is Not-CS tier.

Contains concept definitions only — no cadence thresholds, no playbook actions, no internal CS operating detail.

---

## Gainsight concepts

- **CTA (Call to Action)** — an action item or issue assigned to a CSM. Lives in the Cockpit. Has a type (e.g., Lifecycle, Risk, Opportunity), priority, status, and owner. May contain sub-tasks.

- **Cockpit** — the CSM workspace in Gainsight where CTAs are managed.

- **Success Plan** — a multi-step strategic plan for a customer account. Tracks objectives and percentage complete.

- **Objective** — a line item within a Success Plan.

- **Scorecard** — Gainsight's health scoring system for a customer account. Score from 0–100, labeled A–F.

- **Measure** — a component of a scorecard (e.g., product adoption, support tickets, engagement).

- **Timeline** — the activity log for a customer account. Contains both CSM-logged entries (meetings, emails) and system-generated events (milestones, stage changes).

- **Milestone** — a system-generated timeline event (e.g., contract signed, onboarding started).

- **Company vs Relationship** — a Company is the top-level account (e.g., Acme Inc). A Relationship is a sub-product or sub-entity under the company (e.g., Acme Inc - Mac Suite).

- **GSID** — Gainsight's internal unique identifier for records. A 36-character ID.

- **Customer Category** — a segmentation field that groups accounts by engagement priority and touch model. Values vary by company; Jamf uses Spotlight, Retain, Grow, and React.

- **CSM Sentiment** — the CSM's manual rating of a customer's health and relationship quality. Typically graded A–F.

- **Renewal Status** — tracks where an account is in the renewal process (e.g., open, escalated, notice of churn).

- **ARR (Annual Recurring Revenue)** — the annualized contract value for a customer account. Treated as confidential at Jamf.

---

## Staircase AI concepts

- **Signal** — a risk or opportunity indicator identified by Staircase AI from customer communications (emails, meetings, etc.).

- **Evidence** — the source communications supporting a signal. Includes email or meeting excerpts with links.

- **"No reply found"** — Staircase has no signal for this account or query. This is not an error.

---

## Field naming (Gainsight)

| Suffix | Meaning |
|--------|---------|
| `__gc` | Custom field (Gainsight Custom) |
| `__gr` | Lookup — `Field__gr.Name` resolves to a related record's name |
| `_PicklistLabel` | Auto-companion returning the human-readable label for a PICKLIST field |

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle; rebranded, CalVer, examples genericized, carve-out applied.*
