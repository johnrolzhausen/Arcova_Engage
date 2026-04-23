# Data Model Specification
### Arcova Engage | Power Platform Portfolio Project

---

## Document Control

| Field | Detail |
|---|---|
| **Version** | 1.0 |
| **Scope** | All Dataverse tables, columns, relationships, and choice sets for Arcova Engage v1.0 |

---

## Table Inventory

| # | Table Display Name | Schema Name | Type | Purpose |
|---|---|---|---|---|
| 1 | Account | **account** | Standard | Client organisations |
| 2 | Contact | **contact** | Standard | Individual client contacts |
| 3 | SLA Configuration | **arc_slaconfig** | Custom | SLA hours by category and priority |
| 4 | Engagement | **arc_engagement** | Custom | Core case/project record |
| 5 | Deliverable | **arc_deliverable** | Custom | Outputs and milestones per engagement |
| 6 | Communication | **arc_communication** | Custom | Interaction log per engagement |

---

## Shared Choice Sets (Global Option Sets)

### arc_EngagementStatus

| Label | Value | Description |
|---|---|---|
| New | 100000000 | Submitted but not yet reviewed by an agent |
| In Triage | 100000001 | Under initial review and categorisation |
| Active | 100000002 | Assigned and work is in progress |
| On Hold | 100000003 | Paused - awaiting client input or internal blocker |
| Escalated | 100000004 | Flagged as urgent; senior agent involved |
| Closed | 100000005 | Resolved and closed |

### arc_Priority

| Label | Value | Description |
|---|---|---|
| Low | 100000000 | Standard SLA applies; no urgency |
| Medium | 100000001 | Default priority for most engagements |
| High | 100000002 | Compressed SLA; proactive agent contact required |
| Critical | 100000003 | Immediate response required; escalation likely |

### arc_EngagementCategory

| Label | Value | Description |
|---|---|---|
| Advisory | 100000000 | Strategic guidance and recommendations |
| Technology | 100000001 | Technology implementation or assessment |
| Transformation | 100000002 | Organisational or process change programmes |
| Compliance | 100000003 | Regulatory or audit-related work |
| Other | 100000004 | Does not fit a defined category |

### arc_DeliverableStatus

| Label | Value | Description |
|---|---|---|
| Not Started | 100000000 | Work has not yet begun |
| In Progress | 100000001 | Actively being worked on |
| Under Review | 100000002 | Complete but pending internal or client review |
| Delivered | 100000003 | Accepted and handed to client |
| Cancelled | 100000004 | Removed from scope |

### arc_DeliverableType

| Label | Value | Description |
|---|---|---|
| Report | 100000000 | Written analysis or findings document |
| Workshop | 100000001 | Facilitated session or training |
| Presentation | 100000002 | Slide deck or executive briefing |
| Assessment | 100000003 | Scored or scored evaluation output |
| Roadmap | 100000004 | Strategic plan or phased timeline |
| Other | 100000005 | Custom deliverable type |

### arc_CommunicationDirection

| Label | Value | Description |
|---|---|---|
| Inbound | 100000000 | Client initiated the interaction |
| Outbound | 100000001 | Arcova agent initiated the interaction |

### arc_CommunicationChannel

| Label | Value | Description |
|---|---|---|
| Portal | 100000000 | Submitted via the Canvas App |
| Copilot | 100000001 | Interaction via Arcova Assist agent |
| Email | 100000002 | Email outside the portal |
| Teams | 100000003 | Microsoft Teams message |
| Phone | 100000004 | Phone call (manually logged) |

---

## Table 1 - Account (Standard)

**Schema name**: **account**  
**Type**: Standard (exists in Dataverse by default)  
**Action**: Add to solution. Add one custom column.

### Columns to Use (Standard - No Changes)

| Display Name | Schema Name | Type | Notes |
|---|---|---|---|
| Account Name | **name** | Text | Company name - primary column |
| Industry | **industrycode** | Choice (standard) | Microsoft's built-in industry list |
| Website | **websiteurl** | URL | Optional |

### Custom Column to Add

| Display Name | Schema Name | Type | Required | Notes |
|---|---|---|---|---|
| Client Tier | **arc_clienttier** | Choice | No | Gold / Silver / Bronze |

**arc_ClientTier choice values:**

| Label | Value |
|---|---|
| Bronze | 100000000 |
| Silver | 100000001 |
| Gold | 100000002 |

---

## Table 2 - Contact (Standard)

**Schema name**: **contact**  
**Type**: Standard (exists in Dataverse by default)  
**Action**: Add to solution. No custom columns required in v1.0.

### Columns to Use (Standard - No Changes)

| Display Name | Schema Name | Type | Notes |
|---|---|---|---|
| First Name | **firstname** | Text | |
| Last Name | **lastname** | Text | Primary column is full name |
| Email | **emailaddress1** | Email | Used by flows to send notifications |
| Company Name | **parentcustomerid** | Lookup -> Account | Links contact to their organisation |
| Job Title | **jobtitle** | Text | Contextual for agents |

---

## Table 3 - SLA Configuration (Custom)

**Schema name**: **arc_slaconfig**  
**Type**: Custom  
**Primary column**: **arc_name** (auto-created, display name: "SLA Name")  
**Purpose**: A reference table storing the response and resolution hour targets for each engagement category and priority combination. 

### Columns

| Display Name | Schema Name | Type | Required | Notes |
|---|---|---|---|---|
| SLA Name | **arc_name** | Text | Yes | Auto-created primary column. Use format: "[Category] - [Priority]" e.g. "Advisory - High" |
| Category | **arc_category** | Choice (arc_EngagementCategory) | Yes | Which category this SLA applies to |
| Priority | **arc_priority** | Choice (arc_Priority) | Yes | Which priority tier this SLA applies to |
| Response Hours | **arc_responsehours** | Whole Number | Yes | Hours from submission to first agent contact |
| Resolution Hours | **arc_resolutionhours** | Whole Number | Yes | Hours from submission to closure target |

### Sample Data 

| SLA Name | Category | Priority | Response Hrs | Resolution Hrs |
|---|---|---|---|---|
| Advisory - Low | Advisory | Low | 48 | 240 |
| Advisory - Medium | Advisory | Medium | 24 | 120 |
| Advisory - High | Advisory | High | 8 | 48 |
| Advisory - Critical | Advisory | Critical | 2 | 24 |
| Technology - Medium | Technology | Medium | 24 | 96 |
| Technology - High | Technology | High | 4 | 48 |

---

## Table 4 - Engagement (Custom) Core Table

**Schema name**: **arc_engagement**  
**Type**: Custom  
**Primary column**: **arc_name** (auto-created, rename display to "Engagement Title")  
**Purpose**: The central record representing a client engagement request from submission through to closure.

### Columns

| Display Name | Schema Name | Type | Required | Notes |
|---|---|---|---|---|
| Engagement Title | **arc_name** | Text | Yes | Auto-created primary column |
| Status | **arc_status** | Choice (arc_EngagementStatus) | Yes | Default: New |
| Priority | **arc_priority** | Choice (arc_Priority) | Yes | Default: Medium |
| Category | **arc_category** | Choice (arc_EngagementCategory) | Yes | |
| Description | **arc_description** | Multiline Text | Yes | Client's description of the request |
| Requester | **arc_requester** | Lookup -> Contact | Yes | The individual who submitted the request |
| Client Organisation | **arc_account** | Lookup -> Account | Yes | The company the requester belongs to |
| Assigned Agent | **arc_assignedagent** | Lookup -> User (SystemUser) | No | Populated by intake flow or manual assignment |
| Submitted Date | **arc_submitteddate** | Date and Time | Yes | Set automatically on creation (default: Now) |
| SLA Due Date | **arc_sladuedate** | Date and Time | No | Calculated by intake flow based on arc_slaconfig |
| Closed Date | **arc_closeddate** | Date and Time | No | Set when status changes to Closed |
| Escalated | **arc_escalated** | Yes/No | Yes | Default: No. Set to Yes by Copilot escalation flow |
| Escalation Reason | **arc_escalationreason** | Multiline Text | No | Populated when arc_escalated = Yes |
| Requester Notes | **arc_requesternotes** | Multiline Text | No | Additional context from the requester at intake |

> **Column-by-column design notes:**

> **arc_requester (Lookup -> Contact) vs arc_assignedagent (Lookup -> User)**:

> **arc_sladuedate (not a formula column)**

> **arc_escalated (Yes/No column)**

---

## Table 5 - Deliverable (Custom)

**Schema name**: **arc_deliverable**  
**Type**: Custom  
**Primary column**: **arc_name** (auto-created, display name: "Deliverable Name")  
**Purpose**: An output, milestone, or workstream item linked to a parent engagement.

### Columns

| Display Name | Schema Name | Type | Required | Notes |
|---|---|---|---|---|
| Deliverable Name | **arc_name** | Text | Yes | Auto-created primary column |
| Engagement | **arc_engagement** | Lookup -> arc_engagement | Yes | Parent engagement - this is the key relationship |
| Type | **arc_type** | Choice (arc_DeliverableType) | Yes | |
| Status | **arc_deliverablestatus** | Choice (arc_DeliverableStatus) | Yes | Default: Not Started |
| Due Date | **arc_duedate** | Date Only | No | |
| Assigned To | **arc_assignedto** | Lookup -> User (SystemUser) | No | Which agent owns this deliverable |
| Notes | **arc_notes** | Multiline Text | No | Internal working notes |


### Relationship

- **arc_deliverable N:1 arc_engagement** (many deliverables belong to one engagement)

---

## Table 6 - Communication (Custom)

**Schema name**: **arc_communication**  
**Type**: Custom  
**Primary column**: **arc_name** (auto-created, display name: "Communication Subject")  
**Purpose**: A chronological log of all interactions (portal submissions, Copilot conversations, agent notes, email records) linked to a parent engagement.

### Columns

| Display Name | Schema Name | Type | Required | Notes |
|---|---|---|---|---|
| Communication Subject | **arc_name** | Text | Yes | Brief description of the interaction |
| Engagement | **arc_engagement** | Lookup -> arc_engagement | Yes | Parent engagement |
| Direction | **arc_direction** | Choice (arc_CommunicationDirection) | Yes | Inbound / Outbound |
| Channel | **arc_channel** | Choice (arc_CommunicationChannel) | Yes | Where the interaction occurred |
| Content | **arc_content** | Multiline Text | Yes | The actual message or note content |
| Timestamp | **arc_timestamp** | Date and Time | Yes | When the interaction occurred. Default: Now |
| Contact | **arc_contact** | Lookup -> Contact | No | The client contact involved |
| Agent | **arc_agent** | Lookup -> User (SystemUser) | No | The Arcova agent involved |


### Relationship

- **arc_communication N:1 arc_engagement** (many communications belong to one engagement)
- Relationship behaviour: **Parental** (cascade delete)

---

## Relationship Map

******
Account (1) ──────────────────── (N) arc_engagement
Contact (1) ──────────────────── (N) arc_engagement   [as Requester]
User (1) ─────────────────────── (N) arc_engagement   [as Assigned Agent]

arc_engagement (1) ───────────── (N) arc_deliverable
arc_engagement (1) ───────────── (N) arc_communication

Contact (1) ──────────────────── (N) arc_communication [as Contact]
User (1) ─────────────────────── (N) arc_communication [as Agent]

arc_slaconfig ─────── (referenced by flows, no Dataverse relationship)
******

---

## Phase 0 Schema Checklist

### Tables
- [x] **arc_slaconfig** created inside solution
- [x] **arc_engagement** created inside solution
- [x] **arc_deliverable** created inside solution
- [x] **arc_communication** created inside solution
- [x] **account** (standard) added to solution
- [x] **contact** (standard) added to solution

### Global Option Sets
- [x] **arc_EngagementStatus** created with 6 values
- [x] **arc_Priority** created with 4 values
- [x] **arc_EngagementCategory** created with 5 values
- [x] **arc_DeliverableStatus** created with 5 values
- [x] **arc_DeliverableType** created with 6 values
- [x] **arc_CommunicationDirection** created with 2 values
- [x] **arc_CommunicationChannel** created with 5 values
- [x] **arc_ClientTier** created with 3 values (used on Account)

### Columns
- [x] All **arc_engagement** columns present (13 columns + primary)
- [x] All **arc_deliverable** columns present (7 columns + primary)
- [x] All **arc_communication** columns present (8 columns + primary)
- [x] All **arc_slaconfig** columns present (4 columns + primary)
- [x] **arc_clienttier** added to **account**

### Relationships
- [x] arc_engagement -> Contact (Requester) N:1 configured
- [x] arc_engagement -> Account N:1 configured
- [x] arc_engagement -> User (Assigned Agent) N:1 configured
- [x] arc_deliverable -> arc_engagement N:1 (Parental) configured
- [x] arc_communication -> arc_engagement N:1 (Parental) configured

### Sample Data
- [x] At least 6 rows inserted into arc_slaconfig
- [x] At least 2 Account records created (test clients)
- [x] At least 2 Contact records created (linked to Accounts)
