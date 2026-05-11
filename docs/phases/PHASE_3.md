# Phase 3 - Power Automate Flows

**Status:** Complete
**Build dates:** May 2026
**Solution:** Arcova Engage
**Environment:** Arcova Engage - Dev
**Publisher prefix:** `arc`

## Phase summary

Phase 3 built the **automation backbone** of the Arcova Engage solution: seven Power Automate flows that operate behind the Canvas App (Phase 2) and Model-Driven Agent Hub (Phase 1) to handle data hygiene, business communications, SLA enforcement, lifecycle approvals, and external system integration. Together with the Dataverse schema (Phase 0) and the two front-end apps (Phases 1 and 2), Phase 3 closes the loop on the core platform - a single user action now triggers a coherent chain of automated behaviors across the system.

The phase progressed from the simplest flows (single-action auto-population of timestamps) through increasingly complex patterns (event-driven notifications, scheduled SLA monitoring, asynchronous human approval, externally-callable HTTP endpoints), culminating in a verified end-to-end test where one Canvas App submission cascaded through all seven flows working in concert.


### Flow inventory

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TRIGGER TYPES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Event-driven (Dataverse Added/Modified):                           │
│    Section 2 → Auto-populate Submitted Date                         │
│    Section 3 → Auto-populate Time Stamp                             │
│    Section 4 → Status Change Notify                                 │
│    Section 5 → Engagement Intake Router                             │
│                                                                     │
│  Schedule-driven (hourly recurrence):                               │
│    Section 6 → SLA Breach Alert                                     │
│                                                                     │
│  Manually-triggered (Power Apps V2):                                │
│    Section 7 → Closure Approval Request                             │
│                                                                     │
│  Externally-triggered (HTTP request):                               │
│    Section 8 → Escalation Notifier                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Connection ownership pattern

A consistent ownership model was applied across all seven flows, distinguishing **system actions** from **user-facing actions**:

| Connection type | Owner | Rationale |
|---|---|---|
| Microsoft Dataverse | Service account | Service identity for system-level read/write; survives departure of individual users |
| Office 365 Outlook (shared mailbox) | John (with Send As permission) | User identity required for Exchange authentication; shared mailbox masks individual sender |
| Microsoft Teams | John | User identity required for Teams licensing model |
| Approvals | John | User identity required for the Approvals service |

Every flow is **owned by the service account with John added as co-owner**. This means the flow runs under service identity in production, but John retains edit access for maintenance.

## Section 1 - Foundation

Section 1 established the conditions for Phase 3 work without building any flows. Eight scoping decisions were locked in before any technical work began:

1. **Service account creation.** A free `service_account@rolzhausen.com` Microsoft 365 account with Power Apps for Developer license, providing system identity without consuming a paid seat.
2. **Two-persona structure.** John as the primary agent identity, Cindy added as a second Arcova Agent for cross-assignment testing and approval flows.
3. **Manager email via environment variable.** The `arc_ManagerEmail` environment variable was created to externalize the manager contact, enabling per-environment configuration without code changes.
4. **Approval and acknowledgment patterns.** A formal closure approval pattern (Section 7) for engagement lifecycle gates, and a lightweight acknowledgment pattern (Section 8) for escalation notifications.
5. **Notification matrix.** Per the Project Brief, a defined matrix of which audiences receive which notifications across the engagement lifecycle.
6. **Curated status transitions.** Section 4's status change notifications fire on a curated subset of transitions (Active, On Hold, Escalated, Closed) rather than all status changes, avoiding notification noise on internal-only state moves.
7. **Audit trail discipline.** Every flow writes at least one Communication record to provide a complete audit history; this discipline is enforced at design time, not assumed.
8. **Test harness pattern.** TEST-prefixed records (e.g., "TEST - Acme Corp", "Test Requester") used throughout development; production records remain untouched during iteration.

Section 1 also established the browser session isolation pattern - Firefox dedicated to service account sessions, Edge/Chrome reserved for John's identity - after an initial OAuth-failed-in-incognito issue was diagnosed and resolved.

**Naming convention adopted:** `[Arcova Engage] [Object] - [Action]` for every flow.

## Section 2 - Auto-populate Submitted Date

**Purpose:** Automatically populate `arc_submitteddate` on every newly-created engagement, so the Canvas App and Agent Hub no longer need to set this field manually.

**Trigger:** When a row is added on Engagements, Organization scope.

**Trigger condition:** `@equals(triggerOutputs()?['body/arc_submitteddate'], null)` enforces idempotency - the flow only fires when the field is unset, preventing it from re-firing if the row is updated later. This is the first defensive pattern of Phase 3.

**Action:** Update a row sets `arc_submitteddate` to `utcNow()`.

A subtle but important lesson surfaced here: **trigger conditions evaluated false do not appear in run history**. Flows short-circuit before run creation, which is correct behavior but makes initial debugging confusing - a missing run looks like a missing trigger registration.

## Section 3 - Auto-populate Time Stamp

**Purpose:** Mirror of Section 2, applied to Communication records. Every Communication row, regardless of which flow or app creates it, gets its Time Stamp populated by this flow.

**Trigger:** When a row is added on Communications.
**Trigger condition:** `@equals(triggerOutputs()?['body/arc_timestamp'], null)`
**Action:** Update a row sets `arc_timestamp` to `utcNow()`.

Section 3's significance is structural: it means **every other flow can leave Time Stamp blank** when creating Communications, knowing Section 3 will fill it in. This is the first concrete example of cross-flow composition - a downstream flow guarantees a property of any upstream flow's output.

## Section 4 - Status Change Notify

**Purpose:** When an engagement's status changes to a notification-worthy state (Active, On Hold, Escalated, Closed), email the requester and log the notification.

**Trigger:** When a row is modified on Engagements.
**Select columns:** `arc_status,arc_name,arc_requester,arc_engagementid` (this connector version uses the column-select field as both projection *and* filter - only modifications touching listed columns fire the trigger).
**Trigger condition:** `@contains(createArray(750740002, 750740003, 750740004, 750740005), triggerOutputs()?['body/arc_status'])`

The four numeric values correspond to the curated transition subset. Note the values are **750740000-series** because they were generated against the `arc` publisher prefix - *not* the generic 100000000-series that some Microsoft documentation suggests.

**Actions:** Get Requester Contact → Send Email from shared mailbox → Create audit Communication.

A reusable email template was developed at `flows/email-template-base.html` with branded HTML, navy `#0f3060` header for standard notifications, amber `#B85C00` for SLA breach, and red `#A4262C` for escalation. The template uses `{{TOKEN}}` placeholder syntax for dynamic content injection.


## Section 5 - Engagement Intake Router

**Purpose:** When an engagement is created, automatically calculate its SLA Due Date and assign an agent based on category, then send an intake confirmation email.

This is the most complex flow in Phase 3 and the first that genuinely demonstrates **reference data as configuration**. Two reference tables drive the logic:

- `arc_slaconfig` - combinations of category × priority → resolution hours
- `arc_categoryassignment` - categories → default agent

These tables were seeded as part of Section 5:

| Category | Priority | Resolution Hours | Default Agent |
|---|---|---|---|
| Advisory | Critical/High/Med/Low | 4/24/48/96 | John |
| Technology | Critical/High/Med/Low | 4/24/48/96 | John |
| Compliance | Critical/High/Med/Low | 4/24/48/96 | Cindy |
| Transformation | Critical/High/Med/Low | 4/24/48/96 | Cindy |
| Other | Critical/High/Med/Low | 8/48/72/120 | John |

**Flow structure:** Trigger (Added on Engagements) → Get SLA Config (List rows with OData filter) → Condition (length > 0 - defensive fallback for unseeded combinations) → Compose Resolution Hours → Compose Submitted Date (`utcNow()`) → Calculate SLA Due Date (`addHours()`) → Get Default Agent (List rows on assignment table) → Update Engagement (SLA Due Date + Assigned Agent via lookup binding) → Get Requester Contact → Send intake email → Create intake audit Communication.

The lookup binding for the assigned agent uses the canonical pattern:
```
"arc_assignedagent@odata.bind": "/systemusers(<GUID>)"
```


## Section 6 - SLA Breach Alert

**Purpose:** On a scheduled cadence, identify engagements whose SLA Due Date has passed without resolution, then alert the assigned agent and manager.

**Trigger:** Recurrence (hourly cadence in production; 5-minute cadence during testing for faster iteration).

**Pattern shift:** Section 6 is the first scheduled flow of Phase 3. Event-driven flows are reactive; scheduled flows are proactive. The architectural implications are significant - scheduled flows must implement **idempotency mechanisms** because they fire whether or not work needs to be done.

A schema column was added to support this: `arc_slabreachnotified` (Yes/No, default No). This flag prevents the same breach from being re-notified on every hourly run.

**Get Breaching Engagements:** A List rows action with a four-clause OData filter:
```
arc_sladuedate lt @{utcNow()} 
and arc_status ne 750740005 
and arc_slabreachnotified eq false 
and _arc_assignedagent_value ne null
```

The clauses respectively filter for: past-due, not closed, not already notified, and has an assigned agent.

**Apply to each loop:** For each breaching engagement, get the assigned agent, post a Teams adaptive card alert, send an amber-themed email (To = agent, CC = manager from `arc_ManagerEmail` environment variable), create an audit Communication, and finally mark the engagement as notified.


## Section 7 - Closure Approval Request

**Purpose:** Allow an agent to request manager approval to close an engagement. On approval, the engagement is closed and the audit trail records the approval; on rejection, the engagement remains open and the rejection reason is captured.

**Trigger:** Power Apps (V2) with EngagementId as text input parameter. Section 7 introduces the **manually-triggered flow** pattern - the flow runs only when invoked by a user action, contrasted with event-triggered (Sections 2-5) and scheduled (Section 6).

**Critical new concept: asynchronous wait.** The "Start and wait for an approval" action pauses the flow run until the approver clicks Approve or Reject. Flow runs can persist for minutes, hours, or days. During the wait, the run state is "Running" but no compute is consumed.

**Setup phase:** Get Engagement, Get Requester Contact, Get Assigned Agent, Get Manager Email (Compose from environment variable).

**Approval action:** "Start and wait for an approval", type "Approve/Reject - First to respond". Title and Details populated with engagement context; Assigned to = the manager email. The approval renders as an adaptive card in Teams Approvals.

**Outcome branching:** A Condition action compares `outputs('Start_and_wait_for_an_approval')?['body/outcome']` against the literal string "Approve".

- **Approve branch:** Update engagement to Status = Closed and Closed Date = `utcNow()`, create approval audit Communication (Inbound/Teams direction), notify agent via Teams.
- **Reject branch:** Capture rejection comments via Compose extracting `first(...)?['comments']` (with `empty()` guard for missing comments), create rejection audit Communication, notify agent via Teams with the rejection reason.

### Cross-flow composition demonstrated

The Approve branch updates the engagement's status to Closed, which is one of Section 4's curated notification statuses. Section 4 therefore fires automatically downstream, sending the requester a status-changed email and creating its own audit Communication. The net result of a single Approve click is **multiple Communications** - one for the approval itself, one for the resulting status change, with Section 3 populating Time Stamps on both.


## Section 8 - Escalation Notifier

**Purpose:** Provide an externally-callable endpoint that can mark an engagement as escalated and notify the senior agent. Designed to be invoked by Phase 4's Copilot Studio agent.

**Trigger:** When an HTTP request is received. Section 8 introduces the **externally-triggered flow** pattern - anything that can make an HTTPS POST and possesses the URL can invoke this flow.

**Trigger configuration:**
- JSON schema accepts two string properties: `engagementId` and `escalationReason`
- HTTP method: POST
- "Who can trigger the flow?": **Anyone** (URL-only authentication; see lessons learned below)

**Defensive validation:** A `Validate Inputs` condition gates the flow body, checking `empty(triggerBody()?['engagementId'])`. The If no branch returns HTTP 400 with an error JSON; only valid inputs reach the main flow logic.

**Main flow:** Get Engagement → Get Assigned Agent → Get Senior Agent Email (Compose from environment variable) → Escalate Engagement (sets `arc_escalated = Yes`, `arc_escalationreason`, status = Escalated) → Post adaptive card to Teams (Acknowledge/Decline buttons) → Send red-themed email → Create escalation audit Communication (Channel = Copilot, anticipating Phase 4) → Return HTTP 200 with success JSON.

**Adaptive card JSON:** Section 8 uses the first full adaptive card JSON in the project, with Container styling, FactSet for engagement context, and Action.Submit buttons. The buttons are decorative at this phase - they would invoke a future response-handler flow in a production deployment.

**Testing pattern:** A REST Client template was created at `flows/escalation/escalation-notifier-test.http` with placeholder URL and engagement GUID. The file documents the API contract; the populated version (with the real URL) stays local. The flow URL is treated as a secret because its signature parameter is the authentication.


## Section 9 - Integration, Cleanup, and Documentation

### 9.1 Canvas App cleanup

Phase 2's Submit screen and Engagement Detail's Add Note dialog explicitly populated `arc_submitteddate` and `arc_timestamp` via `Now()`. Sections 2 and 3 now own those values. Leaving the explicit calls in place would create two sources of truth competing for the same fields.

The cleanup removed:

- `'Submitted Date': Now()` from the Submit screen's Patch operation
- `'Time Stamp': Now()` from the Add Note dialog's Patch operation

After publication, a test submission verified that both fields are populated by their respective flows (Section 2 and Section 3), confirming the architectural separation is now fully realized.

### 9.2 Closure Approval Button Surfacing - Deferred to Phase 5

Section 7's flow uses the Power Apps (V2) trigger, which is the modern, parameterized pattern but **does not auto-surface** in a model-driven app's Flow menu. (Only the legacy "When Power Apps calls a flow" trigger gets that free integration.) Surfacing the flow as a production-grade button on the Agent Hub command bar requires modern command designer work - a Power Fx-bound command invoking the flow with the current record's GUID.



### 9.3 End-to-end test scenario

A single Canvas App submission was used to exercise all seven flows in a coherent narrative:

1. Canvas App submission with Category = Technology, Priority = High
2. Section 2 populated Submitted Date
3. Section 5 (Intake Router) routed to John (Technology → John per assignment table), calculated SLA Due Date 24 hours out, sent intake confirmation email, created intake audit Communication
4. Section 3 populated Time Stamp on the intake Communication
5. Agent transitioned status: New → In Triage → Active
6. Section 4 fired on the Active transition, sent status-changed email, created status-change audit Communication
7. Section 3 populated Time Stamp on the status-change Communication
8. SLA Due Date was backdated to simulate breach
9. Section 6 detected the breach on its next run, sent Teams alert, sent amber-themed email (To = John, CC = Cindy), created breach audit Communication, marked the engagement as notified
10. Section 3 populated Time Stamp on the breach Communication
11. Closure Approval flow (Section 7) was invoked via Power Automate Test panel
12. Manager (Cindy) clicked Approve in Teams
13. Section 7 updated status to Closed, created approval audit Communication, notified agent via Teams
14. Section 4 fired downstream on the Closed transition, sent final status-changed email, created another status-change audit Communication
15. Section 3 populated Time Stamps on both new Communications

**The final state:** one engagement, six Communication records, all properly timestamped, telling the complete lifecycle story. The single Canvas App submission produced a coordinated cascade across all seven flows. Cross-flow composition working as designed.

### 9.4 Documentation (this document)

PHASE_3.md captures the architectural decisions, the build progression, the lessons learned, and the verification evidence. PHASE_3.md was deliberately deferred to the end of Phase 3 rather than written incrementally - a practice carried forward from prior project experience where incremental documentation became overwhelming. The README will similarly be deferred to Phase 5 once the full build is complete.

## Schema additions in Phase 3

Two additions to the schema established in Phase 0:

| Column | Table | Type | Default | Purpose |
|---|---|---|---|---|
| `arc_slabreachnotified` | Engagement | Yes/No | No | Section 6 idempotency flag |
| `arc_categoryassignment` | (new table) | - | - | Section 5 category → default agent mapping |

The `arc_categoryassignment` table was created with three columns: Mapping Name (primary), Category (choice), Default Agent (lookup to Users). Five rows were seeded covering all five categories.

## Reference data seeded in Phase 3

| Table | Rows seeded | Purpose |
|---|---|---|
| `arc_slaconfig` | 20 (5 categories × 4 priorities) | Section 5 SLA calculation |
| `arc_categoryassignment` | 5 (one per category) | Section 5 agent routing |




## Verification artifacts

Screenshots committed to `/screenshots/phase3/`:

| Range | Content |
|---|---|
| 01-09 | Section 1 foundation work (service account, environment variable, test harness records) |
| 10-12 | Section 2 build and test |
| 13-15 | Section 3 build and test |
| 16-22 | Section 4 build and test |
| 23-31 | Section 5 build, reference data, and test scenarios |
| 32-36 | Section 6 build and test |
| 37-44 | Section 7 build and test (Approve and Reject scenarios) |
| 45-52 | Section 8 build and test (HTTP trigger, REST Client integration) |
| 53-55 | Section 9.1 Canvas App cleanup |
| 56-63 | Section 9.3 end-to-end test scenario |

Artifacts committed to `/flows/`:

- `email-template-base.html` - Branded HTML template with navy/amber/red header variants for standard, SLA breach, and escalation notifications
- `escalation/escalation-notifier-test.http` - REST Client test template for Section 8's HTTP-triggered flow, with placeholder URL and engagement GUID



---

**Phase 3 complete.** Seven flows built, fully tested, and documented. The autom](63_e2e_full_communications_log.png)