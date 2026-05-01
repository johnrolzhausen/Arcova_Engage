# Phase 1: Agent Hub
### Arcova Engage | Power Platform Portfolio Project

---

## Phase Overview

| Field | Detail |
|---|---|
| **Phase** | 1 - Agent Hub |
| **Status** | Complete - 5/1/2026 |
| **Goal** | Build the model-driven application that Arcova engagement agents use to triage, manage, and resolve client engagements - including security, forms, views, business rules, dashboards, and a guided business process flow on top of the Phase 0 schema |

---

## Step 1 - Custom Security Role

### 1.1 Role Details

Created a custom security role named **Arcova Agent** by copying the **Basic User** role as a baseline.

| Field | Value |
|---|---|
| **Display name** | Arcova Agent |
| **Applies to** | All Users |
| **Description** | Custom security role for Arcova Advisory engagement agents. Grants full read/write access to engagement, deliverable, and communication records, read-only access to SLA configuration, and standard access to Account and Contact records. Does not grant administrative privileges. |

### 1.2 Custom Table Privileges

Org-level access was granted on the three operational tables; SLA Configuration is read-only by design (least-privilege - admins manage SLA rules, agents only consume them).

| Table | Create | Read | Write | Delete | Append | Append To | Assign | Share |
|---|---|---|---|---|---|---|---|---|
| arc_engagement | Org | Org | Org | Org | Org | Org | Org | Org |
| arc_deliverable | Org | Org | Org | Org | Org | Org | Org | Org |
| arc_communication | Org | Org | Org | Org | Org | Org | Org | Org |
| arc_slaconfig | None | Org | None | None | None | Org | None | None |

### 1.3 Standard Table Privileges

| Table | Create | Read | Write | Delete | Append | Append To | Assign | Share |
|---|---|---|---|---|---|---|---|---|
| Account | Org | Org | Org | None | Org | Org | Org | Org |
| Contact | Org | Org | Org | None | Org | Org | Org | Org |

Delete privileges intentionally omitted on Account and Contact - deactivation (which is part of Write) is the correct soft-delete pattern. Permanent deletion of a client organisation or contact would orphan engagements.

### 1.4 Role Assignment

The Arcova Agent role was assigned to the development user account alongside System Administrator (System Administrator retained throughout Phase 1 to enable continued building; production users would have only Arcova Agent).

---

## Step 2 - Forms

### 2.1 Engagement Main Form

Renamed default form from "Information" to "Engagement".

**Header (always visible):**

| Field | Purpose |
|---|---|
| Status | At-a-glance lifecycle position |
| Priority | Urgency indicator |
| Assigned Agent | Ownership |
| SLA Due Date | Deadline pressure |

**Tab structure:**

| Tab | Sections | Components |
|---|---|---|
| Summary | Request Details (2-col), Description (1-col) | 9 fields including Engagement Title, Client Organisation, Requester, Submitted Date, Owner, Category, Priority, Status, Closed Date, Description, Requester Notes |
| Deliverables | Linked Deliverables | Sub-grid bound to Active Deliverables view (later changed to All Deliverables for Engagement - see Section 8D) |
| Communications | Communications Log | Sub-grid bound to Active Communications view |
| Escalation | Escalation Details (2-col) | Escalated, Escalation Reason |

### 2.2 Engagement Quick Create Form

Enabled "Leverage quick-create form if available" on the Engagement table. Built a six-field Quick Create form for fast intake:

| Field | Required |
|---|---|
| Engagement Title | Yes |
| Description | Yes |
| Client Organisation | Yes |
| Requester | Yes |
| Category | Yes |
| Priority | Yes |

Status, SLA Due Date, Assigned Agent, Submitted Date, and Closed Date are intentionally omitted - they're either system-set or populated downstream.

### 2.3 Deliverable Forms

**Main form** with a single General tab containing:

- Deliverable Details section (2-col): Deliverable Name, Engagement, Type, Status, Due Date, Assigned To, Owner
- Notes section (1-col): Notes field at 5 rows for working notes
- Header: Status, Type, Assigned To, Due Date

**Quick Create form** with five fields: Deliverable Name, Engagement, Type, Due Date, Assigned To.

### 2.4 Communication Forms

**Main form** with a single General tab:

- Communication Details section (2-col): Communication Subject, Engagement, Contact, Agent, Direction, Channel, Time Stamp, Owner
- Content section (1-col): Content field for the actual interaction text
- Header: Direction, Channel, Time Stamp, Engagement

**Quick Create form** with five fields: Communication Subject, Engagement, Direction, Channel, Content.

> **Schema correction during build:** The Time Stamp field was initially placed using the system `createdon` column, which is read-only and locked. A custom `arc_timestamp` Date and Time column was created to allow the timestamp to be writable (e.g., backdating a phone call logged later). The same correction was applied to Submitted Date on the Engagement form (using `arc_submitteddate` instead of `createdon`). This aligned the form with the original `DATA_MODEL.md` specification.

### 2.5 SLA Configuration Form

Single tab, single section, five fields: SLA Name, Category, Priority, Response Hours, Resolution Hours, Owner. Quick Create deliberately disabled - admins always use the full form.

### 2.6 Account Form Extension

Added the existing standard Account form to the solution and extended it with the **Client Tier** (`arc_clienttier`) custom column from Phase 0. All other Account fields use Microsoft's standard configuration.

### 2.7 Contact Form Verification

Verified that the standard Contact form contains all fields required by the data model (First Name, Last Name, Email, Company Name, Job Title). No customisations needed.

---

## Step 3 - Views

### 3.1 Engagement Views (6 custom views)

| View | Filter | Sort | Purpose |
|---|---|---|---|
| My Active Engagements | Assigned Agent = Current User AND Status ≠ Closed | SLA Due Date ascending | Personal queue |
| Unassigned Engagements | Assigned Agent is empty AND Status in (New, In Triage) | Submitted Date ascending | Records waiting for pickup |
| Breaching SLA | SLA Due Date older than 1 hour AND Status ≠ Closed | SLA Due Date ascending | Past-due records still open |
| Escalated Engagements | Escalated = Yes AND Status ≠ Closed | SLA Due Date ascending | Records flagged urgent |
| Closed This Week | Status = Closed AND Closed Date in Last 7 Days | Closed Date descending | Recently completed work |
| All Active Engagements | Status ≠ Closed | SLA Due Date ascending | Manager-style overview |

> **Filter design note:** All custom views filter on the `arc_status` custom column rather than the system `statecode` field. Microsoft's default "Active Engagements" view filters on `statecode` (record-level state) which doesn't reflect our business workflow. The two are different concepts and were left to coexist - our views handle the business view; Microsoft's defaults are left untouched.

### 3.2 Deliverable Views (2 views)

| View | Filter | Sort | Used By |
|---|---|---|---|
| Active Deliverables | Status ≠ Delivered AND Status ≠ Cancelled | Due Date ascending | Standalone Deliverables list (in-progress focus) |
| All Deliverables for Engagement | None (no filter) | Due Date ascending | Engagement form's Deliverables sub-grid (full history) |

### 3.3 Communication View

| View | Filter | Sort |
|---|---|---|
| Active Communications | None (no filter; every communication is meaningful log data) | Time Stamp descending |

### 3.4 Quick Find Configuration

Configured the Quick Find views on three custom tables to enable global search across the app:

| Table | Find Columns | View Columns |
|---|---|---|
| Engagement | Engagement Title, Description, Requester Notes, Client Organisation, Requester | Engagement Title, Client Organisation, Status, Priority, SLA Due Date |
| Deliverable | Deliverable Name, Engagement | Deliverable Name, Engagement, Type, Status, Due Date |
| Communication | Communication Subject, Content, Engagement | Communication Subject, Engagement, Direction, Channel, Time Stamp |

> **Default Quick Find filter removed:** Microsoft's default `Status = Active` filter (on the system `statecode` field) was removed from all three Quick Find views. On custom tables with non-standard status workflows, this filter would have either silently excluded records or pointed at a status value that doesn't exist (Deliverable and Communication have no "Active" value).

---

## Step 4 - Business Rule

### 4.1 Require Escalation Reason When Escalated

| Field | Value |
|---|---|
| **Name** | Engagement: Require Escalation Reason When Escalated |
| **Scope** | Entity (Table) - fires both client-side and server-side |
| **Status** | Activated |

**Logic:**

```
IF Escalated equals "Yes"
THEN Set Escalation Reason as Business Required
ELSE Set Escalation Reason as Not Business Required
```

Both branches present - without the ELSE, the field would remain required after Escalated was toggled back to No, blocking saves on records that no longer needed an escalation reason.

> **Scope decision:** Entity scope (rather than Form scope) was chosen so the rule fires regardless of how the record is accessed - via the main form, the Quick Create form, a flow, or the Copilot agent. Form-scoped rules only fire when a specific form is open and would not be enforced by Phase 4's Copilot Studio actions.

### 4.2 Deferred Rules

Two additional business rules were originally planned for Phase 1:

- Auto-populate `arc_submitteddate` to Now() when a new Engagement is created
- Auto-populate `arc_timestamp` to Now() when a new Communication is created

Both were deferred to Phase 3. The classic business rule designer's "Set Field Value" formula type does not support Power Fx-style `Now()` for Date/Time fields. The correct architectural fit is a Power Automate "When a row is added" trigger using `utcNow()` - which is Phase 3 territory and aligns with how the rest of the project's automation will be implemented.

---

## Step 5 - Business Process Flow

### 5.1 Engagement Lifecycle BPF

| Field | Value |
|---|---|
| **Display Name** | Engagement Lifecycle |
| **Schema Name** | arc_engagementlifecycle |
| **Target Table** | Engagement |
| **Status** | Activated |

### 5.2 Stages

| Stage | Required Data Steps |
|---|---|
| New | Engagement Title, Description, Client Organisation, Requester, Category, Priority |
| Triage | Assigned Agent, SLA Due Date |
| Active | Status |
| Resolution | Closed Date |
| Closed | Status (synchronisation checkpoint) |

> **BPF stages and arc_status are independent.** Moving a record forward in the BPF does not automatically change `arc_status` - they are parallel concepts. The BPF guides the user through the lifecycle visually; the user (or a Power Automate flow in Phase 3) updates `arc_status` separately. The "Status" data step in the Closed stage acts as a soft synchronisation, requiring the user to confirm `arc_status = Closed` before formally closing the BPF.

---

## Step 6 - Charts and Dashboards

### 6.1 Charts (3 reusable)

Built on the Engagement table for reuse across both dashboards:

| Chart | Type | Series | Category |
|---|---|---|---|
| Engagements by Status | Pie | Count of Engagements | Status |
| Engagements by Priority | Column | Count of Engagements | Priority |
| Engagements by Category | Pie | Count of Engagements | Category |

> **Charts are context-aware.** A single chart adapts to whichever view it is paired with. The same "Engagements by Status" chart shows different distributions depending on whether it is rendered alongside "My Active Engagements" or "All Active Engagements" - eliminating the need to build duplicate charts for each filter context.

### 6.2 My Workload Dashboard

Personal dashboard for an individual agent. 2x2 layout, all components filtered by **My Active Engagements**:

| Slot | Component | Bound View | Bound Chart |
|---|---|---|---|
| Top-left | List | My Active Engagements (View Selector locked Off) | - |
| Top-right | Chart | My Active Engagements | Engagements by Priority |
| Bottom-left | Chart | My Active Engagements | Engagements by Status |
| Bottom-right | Chart | My Active Engagements | Engagements by Category |

### 6.3 SLA & Triage Health Dashboard

Manager-facing dashboard. 2x2 layout combining urgent operational lists with strategic charts:

| Slot | Component | Bound View | Bound Chart |
|---|---|---|---|
| Top-left | List | Breaching SLA (View Selector locked Off) | - |
| Top-right | List | Unassigned Engagements (View Selector locked Off) | - |
| Bottom-left | Chart | All Active Engagements | Engagements by Priority |
| Bottom-right | Chart | All Active Engagements | Engagements by Status |

> **List View Selector locking.** On both dashboards, the list components have View Selector set to "Off" to prevent users from inadvertently switching the underlying view at runtime. This enforces the dashboard's intent - "My Workload" always shows the user's own work; "SLA & Triage Health" always shows breaching and unassigned records.

---

## Step 7 - Model-Driven App Assembly

### 7.1 App Details

| Field | Value |
|---|---|
| **Display Name** | Arcova Engage Agent Hub |
| **Description** | The primary working application for Arcova engagement agents. Provides triage, engagement management, and reporting capabilities for the agent role. |
| **Designer Used** | Modern App Designer |
| **Status** | Published |

### 7.2 Navigation Structure

Four-group site map organised around Priya's workflow:

```
Arcova Engage Agent Hub
├── Dashboards
│   ├── My Workload
│   └── SLA & Triage Health
├── Work
│   ├── Engagements (all 6 views available)
│   ├── Deliverables
│   └── Communications
├── Clients
│   ├── Accounts
│   └── Contacts
└── Settings
    └── SLA Configuration
```

Most-used items at the top (Dashboards, Work); admin and reference data at the bottom (Settings).

### 7.3 Security Role Association

The Arcova Engage Agent Hub app is associated with the **Arcova Agent** security role (configured via the app's Share dialog). Without this association, users with the role would be unable to launch the app even though they have the underlying table permissions.

---

## Step 8 - Test Data Seeding

Seeded a complete dataset via Excel template imports to make every view, chart, and dashboard populate with realistic data.

### 8.1 Final Record Counts

| Table | Records |
|---|---|
| Account | 5 |
| Contact | 8 |
| Engagement | 20 |
| Deliverable | 31 |
| Communication | 45 |

### 8.2 Engagement Distribution Design

Records were deliberately distributed to populate every custom view:

| Distribution | Count | Populates |
|---|---|---|
| New | 3 | (initial intake records) |
| In Triage | 4 | Unassigned Engagements view |
| Active | 6 | All Active Engagements view |
| On Hold | 2 | (paused work) |
| Escalated | 2 | Escalated Engagements view |
| Closed | 3 | Closed This Week view |
| Unassigned (no agent) | 9 | Unassigned Engagements view |
| Breaching SLA | 3 | Breaching SLA view |
| Critical priority | 4 | Visual emphasis on charts |

### 8.3 Import Method

Excel templates were downloaded from each table's view, then customised via the **Edit Columns** feature on the Download Template dialog to include all columns needed for the import (rather than modifying the underlying view). Data was pasted into the template and imported via the **Import from Excel** action.

> **Import order matters.** Tables were seeded in dependency order: Accounts first (no dependencies), then Contacts (lookup to Account), then Engagements (lookup to Account and Contact), then Deliverables and Communications (lookup to Engagement). Importing in the wrong order would cause lookup resolution failures.

### 8.4 Validation Pass and Issue Resolution

After seeding, every view, dashboard, business rule, and BPF was manually validated against the seeded data. One issue was identified and resolved:

**Issue:** The Engagement form's Deliverables sub-grid was bound to the **Active Deliverables** view, which filters out records with status Delivered or Cancelled. On Closed engagements (where all deliverables had been delivered), the sub-grid appeared empty - giving the false impression that no deliverables existed.

**Resolution:** Created a new view called **All Deliverables for Engagement** with no status filter, and re-bound the Engagement form's Deliverables sub-grid to it. The standalone **Deliverables** entry in the app navigation continues to use Active Deliverables (filtered for in-progress work) - giving each surface a context-appropriate view.

> **AB-410 alignment:** This pattern - using different views for different surfaces of the same table - is a tested concept on the exam. Different audiences and contexts call for different filtering of the same data.

---

## Lessons Learned

- **Always verify breadcrumb / solution context before creating components.** A custom column was once accidentally created outside the solution context and received the default publisher's `cre12_` prefix instead of `arc_`. Schema names cannot be renamed once saved - the column had to be deleted and recreated. Bookmarking the solution URL prevents this.
- **`Now()` is not available in business rule formulas for Date/Time fields.** The classic business rule designer's "Set Field Value" with Formula type only supports date arithmetic on existing fields, not runtime functions. For "set field on creation" logic, use a Power Automate flow with `utcNow()`.
- **The "Edit Columns" button on the Download Template dialog is the right tool for importing fields beyond a view's defaults.** It avoids modifying system views or doing manual entry after the fact.
- **The Modern App Designer's "Add Page" dialog does not include a Group option.** Right-clicking a page (or creating a Group via right-click before adding a page) is the way to create navigation groups.
- **Microsoft's default `statecode = Active` filter on Quick Find views can silently exclude all records on custom tables that use a different status workflow.** Worth removing for any custom table that uses its own status column.
- **Sub-grid views inherit their filters.** When a sub-grid appears empty unexpectedly, the most likely cause is that the bound view's filter excludes the records you expect to see.

---

## Phase 1 Completion Criteria

### Security
- [x] Custom security role `Arcova Agent` created with least-privilege design
- [x] Role assigned to development user
- [x] Role associated with the Arcova Engage Agent Hub app

### Forms
- [x] Engagement main form with header and 4 tabs (Summary, Deliverables, Communications, Escalation)
- [x] Engagement Quick Create form (6 essential fields)
- [x] Deliverable main and Quick Create forms
- [x] Communication main and Quick Create forms
- [x] SLA Configuration main form (no Quick Create)
- [x] Account form extended with Client Tier
- [x] Contact form verified (no customisation needed)
- [x] All forms saved and published

### Views
- [x] 6 custom Engagement views (My Active, Unassigned, Breaching SLA, Escalated, Closed This Week, All Active)
- [x] 2 Deliverable views (Active Deliverables, All Deliverables for Engagement)
- [x] 1 Communication view (Active Communications)
- [x] Quick Find views configured on Engagement, Deliverable, Communication
- [x] Engagement form sub-grids using appropriate views

### Logic
- [x] Business rule "Require Escalation Reason When Escalated" activated at Entity scope
- [x] Business Process Flow "Engagement Lifecycle" with 5 stages activated

### Visualisation
- [x] 3 reusable charts (Engagements by Status, Priority, Category)
- [x] My Workload dashboard (2x2 layout, view-locked)
- [x] SLA & Triage Health dashboard (2x2 layout, view-locked)

### App
- [x] Arcova Engage Agent Hub model-driven app published
- [x] Four navigation groups (Dashboards, Work, Clients, Settings)
- [x] All eight pages added correctly
- [x] App tested at runtime

### Data
- [x] 5 Accounts seeded with full data
- [x] 8 Contacts seeded with Company Name lookups resolved
- [x] 20 Engagements seeded with all lookups resolved
- [x] 31 Deliverables seeded
- [x] 45 Communications seeded
- [x] All seeded records verified via end-to-end validation pass

### Documentation
- [x] Screenshots captured throughout the build (~38 portfolio images committed to `/screenshots/phase1/`)
- [x] `PHASE_1.md` committed to GitHub
