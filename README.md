# vendor-ops

> Operational layer for buy-side procurement: turns vendor contracts into structured data, links MSAs → SOWs → POs, and surfaces the renewal and spend decisions that need action.

**Status:** 🚧 MVP in progress · **Author:** Bob · **Target MVP:** December 2026

**Modules:** Renewal Radar (MVP) · SOW Burn (V1) · Vendor Dedupe (later)

---

## Table of Contents

1. [TL;DR](#1-tldr)
2. [Problem and Evidence](#2-problem-and-evidence)
3. [Users and Jobs to Be Done](#3-users-and-jobs-to-be-done)
4. [Goals, Non-Goals and Metrics](#4-goals-non-goals-and-metrics)
5. [Solution and Requirements](#5-solution-and-requirements)
6. [Scope and Release Plan](#6-scope-and-release-plan)
7. [Technical Considerations](#7-technical-considerations)
8. [Risks, Assumptions and Open Questions](#8-risks-assumptions-and-open-questions)
9. [Alternatives and Decision Log](#9-alternatives-and-decision-log)
10. [Launch and Measurement](#10-launch-and-measurement)
11. [Getting Started](#getting-started)
12. [Data and License](#data-and-license)

---

## 1. TL;DR

Procurement and finance-ops teams make high-stakes decisions on vendor contracts, but the data behind those decisions is fragmented across PDFs, procure-to-pay systems, and spreadsheets.

**vendor-ops** integrates contracts, SOWs and POs into a single **ontology** of vendor objects, then surfaces the decisions that need to be made: which contracts are about to auto-renew, which SOWs are close to exhausting their funding, and which vendors are duplicated.

The MVP, **Renewal Radar**, targets one workflow (renewal decisions) and aims to raise the share of renewals actioned before the notice deadline to **95%+**.

---

## 2. Problem and Evidence

**Problem statement:** Procurement teams can't see their contractual obligations in one place, so renewal and spend decisions are made reactively, often after the window to act has closed.

### Evidence

| Source | What it shows |
| --- | --- |
| Firsthand experience | I managed vendor MSAs and SOWs end to end on the buy side. Renewal terms lived in PDFs, spend lived in the P2P tool, and the link between them lived in people's heads. |
| User interviews (n = _TBD_) | _[Add quotes, e.g. "We found out about the renewal when the invoice arrived."]_ |
| Market signal | P2P systems track transactions; CLM tools track documents. Few connect the two into a single operational picture. |
| Data asset | [CUAD](https://www.atticusprojectai.org/cuad) provides ~500 labeled commercial contracts to validate extraction quality. |

**Cost of doing nothing:** Missed notice windows lock companies into unwanted renewals, lose leverage to renegotiate, and allow SOW overruns that surface only at invoice time.

**Why now:** LLMs can turn unstructured contracts into structured, queryable data at low cost, making a contract-aware decision layer feasible without manual data entry.

---

## 3. Users and Jobs to Be Done

| Persona | Context | Job to be done | Current workaround |
| --- | --- | --- | --- |
| **Primary:** Procurement / vendor manager | Company of 200–2,000 people; owns 50–500 vendor agreements | *When a contract approaches renewal, I want to know early with the key terms in front of me, so I can decide to renew, renegotiate or terminate on my timeline, not the vendor's.* | Spreadsheets, calendar reminders, searching PDFs |
| **Secondary:** Finance-ops / FP&A analyst | Tracks committed vs actual spend | *When a SOW is burning faster than planned, I want an alert before the overrun, so I can adjust forecasts or approve a change order.* | Manual PO-to-invoice reconciliation |
| **Secondary:** Budget owner | Business stakeholder | *I want to know which of my vendors need a decision from me this quarter.* | Emails from procurement |

**Buyer:** Head of Procurement or Controller.

**Not our user (for now):** Legal teams drafting contracts; sell-side contract management.

---

## 4. Goals, Non-Goals and Metrics

### Goals

1. **Operational:** No renewal reaches its notice deadline without a decision from its owner.
2. **Business:** Recover lost negotiating leverage and reduce unplanned spend.
3. **Learning:** Validate that LLM extraction is accurate enough for operators to trust without re-reading the contract.

### Non-Goals

- Contract drafting, redlining or e-signature
- Replacing the procure-to-pay system (integrate, don't compete)
- Full invoice reconciliation

### Metrics

| Type | Metric | Target |
| --- | --- | --- |
| ⭐ North star | % of renewals with an owner decision logged before the notice deadline | ≥ 95% |
| Input | Median lead time between alert and notice deadline | ≥ 30 days |
| Input | % of contracts with an assigned owner | 100% |
| Guardrail | Extraction accuracy on key CUAD clauses (end date, renewal term, notice period) | ≥ 90% |
| Guardrail | False-alert rate | < 10% |

**Kill / pivot criteria:** If extraction accuracy stays below 80% after two iterations, pivot to a human-in-the-loop review queue instead of automated alerts.

---

## 5. Solution and Requirements

### Overview

Model the vendor relationship as connected **objects**, not documents:

- **Object types:** `Vendor`, `MSA`, `SOW`, `PurchaseOrder`, `Clause`, `RenewalEvent`
- **Links:** `Vendor → MSA → SOW → PurchaseOrder`
- **Actions:** Assign owner · Decide (renew / renegotiate / terminate) · Send non-renewal notice · Flag for review

An extraction pipeline turns uploaded contracts into ontology objects. An operational app sits on top, showing users the **decisions due** rather than raw data.

### Core User Flow

1. User uploads MSAs and SOWs.
2. The pipeline extracts key terms with a confidence score; low-confidence fields go to a review queue.
3. The user sees a renewal timeline ordered by notice deadline.
4. The user takes an action (assign, decide, flag); the decision is written back and logged.

### Requirements

| ID | Priority | User story | Acceptance criteria |
| --- | --- | --- | --- |
| R1 | Must | As a vendor manager, I want key renewal terms extracted from each contract so I don't have to read the PDF. | Given an uploaded MSA, the system extracts parties, end date, renewal term and notice period, each with a confidence score. |
| R2 | Must | As a vendor manager, I want a timeline of upcoming notice deadlines. | Contracts within 90 days of a notice deadline appear, sorted by urgency. |
| R3 | Must | As a vendor manager, I want to log a decision on each renewal. | Each decision is stored with owner, date and rationale (audit trail). |
| R4 | Should | As an FP&A analyst, I want SOW burn against committed value. | An alert fires at 80% consumption. |
| R5 | Could | As a procurement lead, I want likely duplicate vendors flagged. | Fuzzy matching surfaces candidate duplicates for review. |
| R6 | Won't (yet) | Direct P2P / ERP integration. | — |

**Non-functional requirements:**
- Contracts treated as sensitive data, with role-based access
- Every extracted field links back to its source clause
- Extraction cost under _[€x]_ per contract

**Edge cases:** Evergreen contracts with no end date · amendments that supersede original terms · scanned PDFs · ambiguous notice clauses. Low-confidence values are routed to human review, never silently trusted.

---

## 6. Scope and Release Plan

| Phase | Target | What ships | Assumption tested | Exit criteria |
| --- | --- | --- | --- | --- |
| **MVP: Renewal Radar** | Dec 2026 | R1–R3 on CUAD + synthetic data | Extraction is accurate enough to trust | ≥ 90% field accuracy; 3 of 4 interviewed operators would use it |
| **V1: SOW Burn** | Feb 2027 | R4 + synthetic PO data | Linking contracts to spend adds decision value | Testers act on at least one burn alert |
| **Later** | — | R5 dedupe, Slack/email alerts, P2P integration | — | — |

**Cut from MVP:** Spend analytics dashboards. Tempting, but descriptive; the MVP should drive decisions, not reporting.

---

## 7. Technical Considerations

```
PDF upload → text extraction (pypdf) → LLM extraction (structured JSON + confidence)
          → ontology data model (Pydantic + SQLite) → Streamlit operational app
```

- **Data:** CUAD for contracts and ground-truth labels; synthetic POs and SOWs generated with Faker.
- **AI/ML:** Per-field evaluation against CUAD labels (precision / recall); confidence thresholds route uncertain fields to human review.
- **Build vs buy:** CLM tools exist, but this project's thesis is the integrated object model across contracts and spend, not document storage.

---

## 8. Risks, Assumptions and Open Questions

### Risks

| Risk type | Risk | Mitigation |
| --- | --- | --- |
| Value | Operators may be "good enough" with spreadsheets | Validate with interviews before V1 |
| Usability | Too many alerts cause fatigue | Rank by deadline and contract value |
| Feasibility | Extraction errors on complex clauses | Field-level confidence + review queue |
| Viability | Sensitive contract data raises security concerns | Access controls; no data retention by LLM provider |

### Key Assumptions

- The notice deadline is the decision that matters most.
- Operators will trust extracted terms if every field links back to its source clause.

### Open Questions

- [ ] How do teams handle contract amendments today?
- [ ] Who owns renewal decisions: procurement or the budget owner?
- [ ] What contract value threshold makes a renewal worth an alert?

---

## 9. Alternatives and Decision Log

### Alternatives Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Do nothing (spreadsheets + calendar) | Free, familiar | Manual, error-prone, blind to SOW burn |
| Extend the P2P tool | Already has spend data | Weak document understanding; slow to configure |
| Standalone CLM tool | Strong on documents | Disconnected from spend |
| **Integrated object model (chosen)** | Connects contracts to spend and decisions | Must build extraction + data model |

### Decision Log

| Date | Decision | Rationale |
| --- | --- | --- |
| 2026-10 | Focus MVP on renewal decisions | Highest-impact, most time-sensitive, and measurable |
| 2026-10 | Defer spend dashboards | Descriptive, not decision-driving |

---

## 10. Launch and Measurement

- **Rollout:** Demo to 3–4 former procurement / finance colleagues, plus classmates for usability testing.
- **Instrumentation:** Log alerts shown, actions taken, and time from alert to decision.
- **Feedback loop:** 15-minute follow-up with each tester; quotes added to this README.
- **Review date:** January 2027, results against accuracy and adoption targets.

---

## Getting Started

> 🚧 Setup instructions coming with the MVP.

```bash
git clone https://github.com/<your-username>/vendor-ops.git
cd vendor-ops
pip install -r requirements.txt
streamlit run app.py
```

---

## Data and License

**Data:** Contract samples and labels from the [Contract Understanding Atticus Dataset (CUAD)](https://www.atticusprojectai.org/cuad) by The Atticus Project, licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Purchase orders and SOW spend data are synthetic.

**License:** Code released under the [MIT License](LICENSE).
