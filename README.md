# Meridian Loan Prescreen

**Performer** | UiPath REFramework | Unattended

The second stage in the Meridian Loan Processing pipeline. This automation consumes applications from the `PrescreenedApplications` Orchestrator queue, retrieves the corresponding application and customer screening data from MongoDB, and runs a three-tier rules engine to determine the outcome: **auto-deny**, **enhanced review**, **auto-approve**, or **standard underwriting**.

---

## Features

- **Queue-Driven Performer** -- Pulls transaction items from the Orchestrator `PrescreenedApplications` queue using the standard REFramework Get Queue Item pattern
- **Kill Rules (Auto-Deny)** -- Evaluates hard-knockout criteria that result in immediate denial with ECOA-defensible reason codes; multiple deny reasons can accumulate per application
- **Enhanced Review Detection** -- Identifies applications that warrant senior underwriter attention based on loan amount, credit score, or financial capacity -- serving business goals beyond just risk
- **Auto-Approve Engine** -- Clears low-risk applications through an 8-point validation gate, approving only when every metric passes
- **Four-Way Routing** -- Each application exits to one of four distinct queues: `DeniedApplications`, `EnhancedReview`, `ApprovedApplications`, or `StandardUnderwriting`
- **MongoDB State Management** -- Updates application status and reason codes in the `applications` collection at each decision point, maintaining a complete audit trail
- **Externalized Thresholds** -- All business rule thresholds are configurable via Config.xlsx and Orchestrator Assets, allowing business analysts to tune criteria without code changes

---

## Process Flow

```
Start
  |
  v
[Get Transaction Item]
  |- Pull from "PrescreenedApplications" queue
  |- Extract AppId from queue item SpecificContent
  |- If no items remain --> End Process
  |
  v
[Fetch Application Data]
  |- MongoDB Find: applications collection by AppId
  |- If not found --> BusinessRuleException
  |- Regex-clean ObjectId wrappers, deserialize to JObject
  |
  v
[Fetch Customer Screening Data]
  |- MongoDB Find: customers collection by customer_id
  |- If not found --> BusinessRuleException
  |- Deserialize screening record (credit, income, debt data)
  |
  v
[Kill Rules] -- Hard Knockouts
  |- Defaults on file >= 1?              --> DENY
  |- Credit score <= threshold?           --> DENY
  |- Debt-to-income ratio >= threshold?   --> DENY
  |- Derogatory marks >= threshold?       --> DENY
  |- Low credit + recent delinquencies?   --> DENY
  |  (Multiple reasons accumulate)
  |
  |- If ANY triggered:
  |     Update MongoDB status = "Denied" + reasons
  |     Add to "DeniedApplications" queue
  |     Return (skip remaining checks)
  |
  v
[Enhanced Review] -- Escalation Triggers
  |- Loan amount > HighValueAmount?       --> ENHANCED (HighValue)
  |- Credit score >= HighScore?           --> ENHANCED (HighScore)
  |- Capacity index > threshold?          --> ENHANCED (HighCapacity)
  |    (annual_income + savings - debt)
  |
  |- If triggered:
  |     Update MongoDB status = "EnhancedReview" + type
  |     Add to "EnhancedReview" queue
  |     Return
  |
  v
[Auto-Approve] -- Low-Risk Gate (ALL must pass)
  |- No defaults on file
  |- No derogatory marks
  |- No delinquencies in last 2 years
  |- Credit score >= 740
  |- DTI ratio <= 0.35
  |- LTI ratio <= 0.30
  |- PTI ratio <= 0.10
  |- Annual income >= $50,000
  |- Loan amount <= $30,000
  |
  |- If ALL pass:
  |     Update MongoDB status = "Approved"
  |     Add to "ApprovedApplications" queue
  |     Return
  |
  v
[Default: Standard Underwriting]
  |- Update MongoDB status = "StandardUnderwriting"
  |- Add to "StandardUnderwriting" queue
  |
  v
[Set Transaction Status] --> Loop
```

---

## Routing Summary

| Outcome | Queue | Trigger | MongoDB Status |
|---|---|---|---|
| Auto-Deny | `DeniedApplications` | Any kill rule fires | `Denied` |
| Enhanced Review | `EnhancedReview` | High value, high score, or high capacity | `EnhancedReview` |
| Auto-Approve | `ApprovedApplications` | All 8+ validation checks pass | `Approved` |
| Standard Underwriting | `StandardUnderwriting` | Default -- no other rule triggered | `StandardUnderwriting` |

---

## Kill Rules Detail

These represent clear, regulator-recognizable knockouts that map to ECOA adverse action reasons:

| Rule | Condition | Denial Reason |
|---|---|---|
| Defaults | `defaults_on_file >= 1` | Applicant has at least one Default on file |
| Credit Score | `pulled_credit_score <= CreditScoreKill` | Applicant credit score is too low |
| DTI Ratio | `debt_to_income_ratio >= DTIKill` | Applicant Debt vs Income is too high |
| Derogatory Marks | `derogatory_marks >= DerogatoryKill` | Too many active derogatory marks on file |
| Compound Risk | `credit_score < 560 AND delinquencies_last_2yrs > 2` | Credit score combined with active delinquency assumes too high risk |

Multiple reasons can accumulate -- a single application may be denied for several independent causes, all recorded.

---

## Enhanced Review Triggers

Enhanced review serves business goals beyond risk mitigation:

| Trigger | Condition | Rationale |
|---|---|---|
| High Value | `loan_amount > HighValueAmount` | Large loans require senior underwriter sign-off |
| High Score | `pulled_credit_score >= HighScore` | Potentially under-priced applications worth a second look |
| High Capacity | `income + savings - debt > CapacityIndex` | Wealth-management cross-sell opportunities |

---

## Tools & Technologies

| Component | Technology |
|---|---|
| RPA Platform | UiPath Studio 26.0 (Windows target) |
| Framework | REFramework (Robotic Enterprise Framework) |
| Database | MongoDB (via InternalLabs.MongoDB.Activities) |
| Queue (Input) | UiPath Orchestrator -- `PrescreenedApplications` |
| Queues (Output) | `DeniedApplications`, `EnhancedReview`, `ApprovedApplications`, `StandardUnderwriting` |
| Config | REFramework Config.xlsx + Orchestrator Assets |

---

## Key Workflows

| File | Purpose |
|---|---|
| `Main.xaml` | REFramework state machine entry point |
| `Framework/GetTransactionData.xaml` | Gets next queue item from `PrescreenedApplications` |
| `Framework/Process.xaml` | Orchestrates the full triage: fetch data, kill rules, enhanced review, auto-approve, route |
| `Meridian/KillRules.xaml` | Evaluates hard-knockout deny criteria; outputs `Denied` flag + reason list |
| `Meridian/EnhancedReview.xaml` | Checks escalation triggers (value, score, capacity); outputs `Enhanced` flag + type |
| `Meridian/AutoApprove.xaml` | 8-point validation gate for low-risk auto-approval |
| `Meridian/Update Denied.xaml` | Helper: sets denied flag and appends reason string to the deny list |

---

## Configuration

| Setting | Purpose |
|---|---|
| `OrchestratorQueueName` | Input queue name (`PrescreenedApplications`) |
| `OrchestratorQueueFolder` | Orchestrator folder path |
| `DBConnection` | MongoDB connection string |
| `Database` | MongoDB database name |
| `CreditScoreKill` | Credit score threshold for auto-deny |
| `DTIKill` | Debt-to-income ratio threshold for auto-deny |
| `DerogatoryKill` | Derogatory marks threshold for auto-deny |
| `HighValueAmount` | Loan amount threshold triggering enhanced review |
| `HighScore` | Credit score threshold triggering enhanced review |
| `CapacityIndex` | Net capacity threshold triggering enhanced review |

---

## Design Decisions

- **Three-tier evaluation order** -- Kill rules run first (cheapest check, highest conviction), then enhanced review, then auto-approve. If no rule fires, the application defaults to standard underwriting. This ensures borderline cases always get human judgment.
- **Accumulated deny reasons** -- Rather than short-circuiting on the first kill rule, all applicable reasons are collected. This mirrors ECOA requirements where adverse action notices must cite all material factors.
- **BusinessRuleException for missing data** -- If an application or customer record can't be found in MongoDB, the transaction throws a BusinessRuleException (skipped, not retried) rather than a system exception, preventing infinite retry loops on data integrity issues.
- **Regex ObjectId cleanup** -- MongoDB's BSON ObjectId wrappers (`ObjectId("...")`) are stripped via regex before JSON deserialization, keeping the data pipeline clean without requiring custom serializers.
- **Externalized thresholds** -- Kill and enhanced review thresholds live in Config.xlsx / Orchestrator Assets. A business analyst can tighten lending criteria during a downturn without filing a code change request.

---

## Pipeline Context

This is **Stage 2** of the Meridian Loan Processing pipeline:

```
[Email Processor] --> [Loan Prescreen] --> [Loan Assigner] --> [Underwriter Dashboard]
     Intake             (this)             HITL Assignment      Attended Workspace
```

The upstream **Email Processor** dispatches items to `PrescreenedApplications`. This performer routes them to one of four downstream queues consumed by the **Loan Assigner** and future notification performers.

---

## Dependencies

```
InternalLabs.MongoDB.Activities  [1.0.1]
UiPath.Excel.Activities          [3.5.1]
UiPath.System.Activities         [26.2.4]
UiPath.Testing.Activities        [25.10.2]
UiPath.UIAutomation.Activities   [25.10.31]
UiPath.WebAPI.Activities         [2.4.0]
```
