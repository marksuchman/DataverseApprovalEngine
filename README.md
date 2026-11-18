# Dataverse Approval Engine

A metadata-driven approval framework for Microsoft Power Platform. One set of flows handles
approvals for **any** Dataverse transaction entity — you onboard a new entity by adding
configuration rows, not by building another flow.

Presented at **DynamicsCon Regional Texas**. The deck is in this repo.

---

## The idea

Most Power Platform approval implementations end up with one flow per entity. Bids get a flow,
contracts get a flow, change orders get a flow, and each one re-implements routing, reminders,
escalation and logging slightly differently.

This framework inverts that. Approval policies, stages, gating rules and approver assignments
live as **Dataverse data**. A single Orchestrator flow reads that configuration at runtime and
drives the approval. Adding approvals to a new entity is an administrative task performed in a
model-driven app.

Design principles that shaped it:

- **Configuration over code** — policies, stages, rules and approvers are rows, not flow logic.
- **Entity-agnostic orchestration** — `Approval Instance` is a custom *Activity* entity, so it
  attaches to any activity-enabled table through the native `Regarding` lookup.
- **Separation of concerns** — Rules decide *whether a stage fires*; Approvers decide *who is
  notified*. They are independent siblings under a Stage.
- **Structured conditions, not expressions** — rules use typed columns (Field Path, Field Type,
  Operator, Value) rather than free-text formulas, with dot-notation traversal to parent records
  resolved in a single OData `$expand`.
- **Implicit ordering** — stages run in Sequence Number order; approvers within a stage are
  always notified in parallel. There is no execution-mode switch to get wrong.

---

## What's in this repo

| File | What it is |
| --- | --- |
| `approvalengine_1_0_0_54.zip` | The **unmanaged** Dataverse solution, v1.0.0.54 |
| `IES_Approval_Framework_Architecture_6.docx` | Architecture design document — data model, flow-by-flow behaviour, security model, worked example |
| `IES_Approval_Framework_Architecture_Addendum_SubgridMechanics.docx` | How the approval subgrids on transaction forms are filtered and rendered |
| `IES_Approval_Framework_IT_Admin_Guide_1.docx` | Step-by-step guide to onboarding a new transaction type |
| `IES_Approval_Framework_IT_Admin_Guide_Addendum_FormEnablement.docx` | Enabling the Approvals tab, subgrids and command-bar buttons on a form |
| `Approval Engine v6 - Bug History.pdf` | The ~35 defects found and fixed while building v6, and how each surfaced |
| `Approval Engine - DynamicsCon Regional Texas.pptx` | Conference deck |

The bug history is worth a read before you build anything similar. Most of those defects
**failed silently rather than erroring**, and nearly all were caught by live tests rather than by
reading the flows.

---

## Data model

Eight tables — six configuration/runtime, one Activity, one audit log.

```
Approval Policy ──1:N──► Approval Stage ──1:N──► Approval Rule    (does this stage fire?)
                                        └─1:N──► Approver         (who gets notified?)

Approval Delegation   (standalone — substitutes delegates for primaries)
Role Holder           (standalone — resolves role-based approvers to users at runtime)

Approval Instance (Activity) ──Regarding──► your transaction record
                             ──1:N────────► Approval Log
```

| Logical name | Purpose |
| --- | --- |
| `ies_approvalpolicy` | One row per transaction type requiring approval |
| `ies_approvalstage` | Ordered steps within a policy |
| `ies_approvalrulecondition` | Typed conditions gating whether a stage fires |
| `ies_approver` | Recipient list for a stage |
| `ies_approvaldelegation` | Time-bounded out-of-office delegation |
| `ies_roleholder` | Maps organisational roles to users |
| `ies_approvalinstance` | Activity entity tracking a live approval |
| `ies_approvallog` | Full audit trail of every approval action |

---

## Flows

| Flow | Role |
| --- | --- |
| `AE6-01 Approval Orchestrator` | Main driver — resolves stages, sends approvals, advances |
| `AE6-00 Condition Evaluation` | Evaluates a single rule condition |
| `AE6-10 Evaluate Stage Rules` | Decides whether a stage fires |
| `AE6-09 Resolve Stage Approvers` | Expands roles, applies delegation and de-duplicates |
| `AE6-02 Reminder and Escalation` | Scheduled; reminders, escalation and hard timeout |
| `AE6-03 Approval Reset` | Restarts an approval mid-stream after a record change |
| `AE6-04 Approval Recall` | Requestor withdraws an in-flight approval |
| `AE6-05 OOO and Undeliverable Monitor` | Detects bounces and out-of-office replies |
| `AE6-06 Notification Resend` | Re-sends a lost Teams card or email |
| `AE6-07 Cancel Approval` | Cancels the underlying Power Automate approval item |

### Why reminders are a separate flow

Power Automate's *Wait for an approval* action suspends the flow. Between stages the
Orchestrator is not running, not polling, and not consuming compute — the platform persists its
state and wakes it when a response arrives. A single run can span weeks of wall-clock time.

Two consequences shaped the architecture:

1. **Time-based logic cannot live in the Orchestrator**, because it is asleep. Reminders and
   escalation run as an independent scheduled flow that queries Dataverse for what is pending.
2. **Stale-response handling is mandatory.** The world changes while the Orchestrator sleeps —
   records get edited, delegations activate, approvals get recalled. The Orchestrator re-reads
   Approval Instance state on *every* resume. Responses arriving against a cancelled or completed
   instance are logged for audit but do not affect the outcome.

Power Automate caps a single run at **30 days**. The framework sets the escalation flow's hard
timeout well below that (14 days recommended), so a long-pending approval is cleanly cancelled
with a `Timed Out` status rather than hitting the platform limit.

---

## Capabilities

- Sequential stages, with parallel approvers inside a stage
- `First to Respond` (any one response completes the stage) or `All Must Approve`
  (any reject terminates it)
- Conditional stage firing via AND/OR condition groups; a stage with no rules always fires
- Different approvers by transaction value, expressed as separate rule-gated stages
- Policy-level self-approval prevention — drops the record owner from any resolved approver list
- Time-bounded delegation; when active, the delegate **replaces** the primary entirely
- Reminders, escalation to a configured target, and hard timeout
- Recall by requestor, and restart mid-stream when a record change invalidates prior approvals
- Notification resend from a command-bar button, without disturbing the underlying approval
- Out-of-office and undeliverable detection, alerting the record owner

---

## Installing

**Prerequisites**

- A Dataverse environment with Power Automate and the Approvals connector
- A publisher using the `ies` prefix, or a willingness to take it on (see caveats)
- System Customizer or System Administrator
- Transaction entities must be **activity-enabled** — the framework attaches through `Regarding`

**Import**

1. Power Apps maker portal → **Solutions** → **Import solution**
2. Select `approvalengine_1_0_0_54.zip`
3. Supply connections for Dataverse, Approvals and Office 365 Outlook when prompted
4. Confirm all ten `AE6-*` flows are **Activated** after import — flows commonly land deactivated
5. Open the **ies_ApprovalEngine** model-driven app to reach the configuration tables

Or with the Power Platform CLI:

```bash
pac solution import \
  --path approvalengine_1_0_0_54.zip \
  --activate-plugins \
  --publish-changes
```

Add `--force-overwrite` when re-importing over existing unmanaged customisations. Without it,
changes to components that already exist are **skipped silently** while the import still reports
success.

---

## Onboarding a new transaction type

Configuration, not development. The IT Admin Guide covers each step in full, with worked
examples and a troubleshooting section.

1. Verify the entity is activity-enabled; document its key fields and related entities
2. Configure Role Holders for any role-based approvers
3. Create the Approval Policy (entity logical name, self-approval flag, record-type filter)
4. Create Approval Stages in sequence order, choosing the Approval Type per stage
5. Create Rules that gate each stage
6. Create Approvers, with escalation targets
7. Build a small trigger flow that creates the Approval Instance
8. Add the Approvals tab, subgrids and command-bar buttons to the entity form
9. Run the smoke test, conditional tests and edge-case tests before promoting

Skip logic is expressed as its inverse. "Skip this stage when Fast Track = Yes" becomes a rule
reading `Fast Track ≠ Yes`.

---

## Caveats

**This is an unmanaged solution extracted from a working implementation, not a product.**
Read it, learn from it, adapt it — but budget time before trusting it in production.

- **Publisher prefix `ies` is baked in.** Tables import as `ies_approvalpolicy` and so on. There
  is no supported way to re-prefix after import; to use your own, rebuild the schema with your
  publisher and port the flow logic.
- **The solution touches `systemuser` and `team`.** Review what it changes before importing into
  an environment you care about.
- **Documentation is written for IES Holdings' environment** and refers to its security roles,
  apps and segments. The architecture generalises; the specifics will not.
- **Documents date from May 2026**; the solution is v6 (October 2026). Where they disagree, the
  solution is authoritative.
- Provided **as-is**, with no warranty and no support commitment.

---

## Licence

[MIT](LICENSE) — Copyright © 2026 IES Holdings, Inc.

Use it, change it, ship it in commercial work, keep the copyright notice. Published with the
approval of IES Holdings.

---

## Credits

Built for the IES Holdings CRM implementation programme. Bug history compiled by Jacob O'Malley.

Questions: [markcsuchman@gmail.com](mailto:markcsuchman@gmail.com) ·
[LinkedIn](https://www.linkedin.com/in/marksuchman/)
