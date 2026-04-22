# Project Brief: Arcova Engage
### A Microsoft Power Platform Portfolio Project

---

## Document Control

| Field | Detail |
|---|---|
| **Document Title** | Arcova Engage - Project Brief |
| **Version** | 1.0 |
| **Author** | John D Rolzhausen |
| **Certification Target** | Microsoft AB-410: Power Platform Developer Associate (Intelligent Applications Builder) |
| **Last Updated** | April 2026 |

---

## 1. Executive Summary

**Arcova Engage** is a Power Platform solution built for **Arcova Advisory**, a fictional mid-size professional services firm that manages consulting engagements for enterprise clients. The firm receives project intake requests through unstructured channels (email, phone), lacks a centralized system for tracking engagement status, and has no mechanism for clients to self-serve information about their projects without contacting their account manager directly.

This solution replaces that friction with a structured, AI-assisted engagement management platform built entirely on the Microsoft Power Platform stack. It serves two distinct audiences

- **Requesters** (clients who submit and track engagements) and 
- **Agents** (Arcova staff who triage, manage, and resolve those engagements) 

through purpose-built apps tailored to each role.

The project is designed as a **portfolio piece and certification vehicle**, demonstrating competency across Power Apps, Power Automate, Dataverse, and Copilot Studio in a coherent, real-world scenario.

---

## 2. The Client

**Arcova Advisory** is a boutique management and technology consulting firm with approximately 200 staff, serving 50–80 active enterprise clients at any given time. Client engagements range from short-term advisory sprints to multi-month transformation programs.

**Industry**: Professional Services / Management Consulting  
**Size**: Mid-market (fictional)  
**Core problem**: High-friction project intake, zero client self-service, manual triage burden on senior staff

---

## 3. Problem Statement

Arcova Advisory currently manages all client engagement requests through a combination of email threads, spreadsheets, and verbal handoffs. This results in:

- **No structured intake**: Requests arrive with inconsistent information, requiring multiple back-and-forth exchanges before work can begin.
- **Zero client visibility**: Clients cannot check the status of their engagements without emailing or calling their account manager.
- **Manual triage overhead**: Agents spend significant time categorising, prioritising, and routing requests that could be automated.
- **Missed SLAs**: Without automated tracking, engagement response and delivery deadlines are frequently missed without warning.
- **Escalation black holes**: Urgent requests that need senior sign-off get lost in email chains with no audit trail.

**The opportunity**: Replace the fragmented process with a structured, automated, and AI-assisted engagement management platform, built on tools Arcova's IT team already has licensed through Microsoft 365.

---

## 4. Proposed Solution

Arcova Engage is a three-surface platform with a shared Dataverse source:

### 4.1 Surface 1: Client Portal (Canvas App)
A clean, mobile-responsive canvas application for **Requesters** (clients). Allows clients to:
- Submit new engagement requests through a structured intake form
- View the real-time status of their active and historical engagements
- Review deliverables attached to their projects
- Interact with the embedded **Arcova Assist** Copilot agent for self-service queries


### 4.2 Surface 2: Agent Hub (Model-Driven App)
A data-dense model-driven application for **Agents** (Arcova staff). Allows agents to:
- View and manage their triage queue with sortable, filterable views
- Update engagement status, priority, and assignment
- Log communications and attach deliverables against cases
- Trigger escalation workflows and approval requests
- Access dashboards showing team workload and SLA health

### 4.3 Surface 3: Arcova Assist (Copilot Studio Agent)
An multi-topic Copilot agent embedded in the Canvas App. Arcova Assist can:
- **Answer status queries** from Dataverse ("What's the status of my Alpha Project engagement?")
- **Accept new engagement submissions** via guided conversation, with data written back to Dataverse
- **Trigger escalation flows** when a client flags urgency, notifying the assigned agent via Teams/email
- **Render adaptive cards** for structured data display (e.g. engagement summary, deliverable list)
- **Use Dataverse as a knowledge source** for FAQ-style queries about engagement policies and process

---

## 5. Technical Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     Power Platform Environment               │
│                                                              │
│  ┌──────────────────┐          ┌──────────────────────────┐  │
│  │  Canvas App      │          │  Model-Driven App        │  │
│  │  (Client Portal) │          │  (Agent Hub)             │  │
│  │  Requesters      │          │  Agents / Staff          │  │
│  └────────┬─────────┘          └───────────┬──────────────┘  │
│           │                               │                  │
│           └──────────────┬────────────────┘                  │
│                          │                                   │
│                ┌─────────▼─────────┐                         │
│                │     Dataverse      │                        │
│                │  Cases / Projects  │                        │
│                │  Contacts          │                        │
│                │  Deliverables      │                        │
│                │  Communications    │                        │
│                │  SLA Config        │                        │
│                └─────────┬─────────┘                         │
│                          │                                   │
│          ┌───────────────┴───────────────┐                   │
│          │                               │                   │
│  ┌───────▼─────────┐            ┌─────────▼────────────┐     │
│  │ Power Automate  │            │  Copilot Studio      │     │
│  │ • Intake routing│            │  Arcova Assist       │     │
│  │ • Notifications │            │  • Status topics     │     │
│  │ • Approvals     │            │  • Submission flow   │     │
│  │ • SLA breach    │            │  • Escalation action │     │
│  │ • Escalation    │            │  • Adaptive cards    │     │
│  └─────────────────┘            │  • Dataverse KB      │     │
│                                 └──────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

**Data flows in one direction through Dataverse**, neither app calls the other directly. Power Automate acts as the process engine. Copilot Studio reads from and writes to Dataverse via connectors and flow-triggered actions. This separation of concerns is intentional and architecturally sound.

---

## 6. Core Dataverse Entities (Proposed)

| Table | Purpose | Key Columns |
|---|---|---|
| `arc_engagement` | The central case/project record | Title, Status, Priority, Category, Requester, Assigned Agent, SLA Due Date |
| `arc_deliverable` | Output or milestone linked to an engagement | Name, Type, Due Date, Status, Linked Engagement |
| `arc_communication` | Log of all client/agent interactions | Direction, Channel, Content, Timestamp, Linked Engagement |
| `arc_sla_config` | SLA rules by engagement category | Category, Response Hours, Resolution Hours |
| `Contact` (standard) | Client contacts | Name, Email, Company (linked to Account) |
| `Account` (standard) | Client organisations | Name, Industry, Tier |


---

## 7. Power Automate Flows (Planned)

| Flow Name | Trigger | Purpose |
|---|---|---|
| `Engagement Intake Router` | New engagement record created | Assigns agent based on category/workload, sends confirmation to requester |
| `SLA Breach Alert` | Scheduled / record condition | Checks overdue engagements and notifies assigned agent + manager |
| `Escalation Notifier` | Copilot Studio → HTTP trigger | Sends Teams message + email to senior agent when escalation flagged |
| `Approval Request` | Agent action in Model-Driven App | Sends adaptive card approval to manager via Teams |
| `Status Change Notify` | Engagement status updated | Emails requester with current status and next steps |

---

## 8. Copilot Studio Agent Topic Map

| Topic | Trigger Phrases | Action |
|---|---|---|
| `Check Engagement Status` | "What's the status of...", "Where are we with..." | Query Dataverse, return adaptive card with engagement summary |
| `Submit New Engagement` | "I'd like to raise a request", "New project" | Guided conversation → write record to Dataverse → trigger intake flow |
| `List My Deliverables` | "What deliverables are due?", "Show me my milestones" | Query Dataverse, return adaptive card list |
| `Escalate an Issue` | "This is urgent", "I need to escalate" | Confirm escalation → trigger Power Automate flow → confirm to user |
| `FAQs / Process Queries` | "How long does intake take?", "What's your SLA?" | Dataverse knowledge source answers |
| `Fallback / Handoff` | Unrecognised intent | Graceful fallback + option to contact agent directly |

---

## 9. User Personas

### Persona 1: The Requester (Client)
**Name**: Jordan Mills, Operations Director at a client firm  
**Goal**: Submit engagement requests quickly, track progress without chasing, get answers without waiting for a callback  
**Pain point**: No visibility, no structure, always feels like requests disappear into a black hole  
**Primary surface**: Canvas App + Copilot agent  
**Technical comfort**: Low-to-moderate (needs an intuitive UI)

### Persona 2: The Agent (Arcova Staff)
**Name**: Priya Nair, Senior Engagement Manager at Arcova Advisory  
**Goal**: Work through a prioritised triage queue efficiently, keep clients updated, hit SLAs  
**Pain point**: Too much time in email, no single source of truth for engagement data  
**Primary surface**: Model-Driven App  
**Technical comfort**: Moderate (comfortable with structured data tools)

---

## 10. Build Phases & Milestones

| Phase | Title | Key Deliverables |
|---|---|---|
| **Phase 0** | Foundation | GitHub repo, environment setup, Dataverse schema, solution setup |
| **Phase 1** | Agent Hub | Model-driven app | views, forms, dashboards, business rules |
| **Phase 2** | Client Portal | Canvas app | intake form, status tracker, deliverables screen |
| **Phase 3** | Automation | All five Power Automate flows built, tested, and documented |
| **Phase 4** | Copilot Agent | Arcova Assist | all six topics, adaptive cards, Dataverse knowledge, flow actions |
| **Phase 5** | Integration & Polish | Embed Copilot in canvas app, end-to-end testing, README finalization |

---

## 11. GitHub Repository Structure (Planned)

```
arcova-engage/
├── README.md                        ← Project overview, demo GIFs, tech stack badges
├── PROJECT_BRIEF.md                 ← This document
├── docs/
│   ├── architecture/
│   │   ├── ARCHITECTURE.md          ← Full architecture narrative
│   │   └── architecture-diagram.png
│   ├── requirements/
│   │   ├── FUNCTIONAL_REQUIREMENTS.md
│   │   └── DATA_MODEL.md
│   ├── personas/
│   │   └── PERSONAS.md
│   └── phases/
│       ├── PHASE_0.md
│       ├── PHASE_1.md
│       ├── PHASE_2.md
│       ├── PHASE_3.md
│       ├── PHASE_4.md
│       └── PHASE_5.md
├── flows/
│   └── [exported flow JSON files]
├── copilot/
│   └── [Copilot Studio export / topic documentation]
└── screenshots/
    └── [app screenshots per phase]
```

---

## 12. AB-410 Certification Alignment

This project is designed to provide direct, demonstrable coverage of the AB-410 exam skill domains:

| AB-410 Skill Domain | Covered By |
|---|---|
| Describe and configure Dataverse tables, columns, and relationships | Phases 0–1 (schema design + model-driven app) |
| Build and configure model-driven apps | Phase 1 (Agent Hub) |
| Build and configure canvas apps | Phase 2 (Client Portal) |
| Create and configure Power Automate cloud flows | Phase 3 (all five flows) |
| Create agents using Copilot Studio | Phase 4 (Arcova Assist) |
| Configure knowledge sources for agents | Phase 4 (Dataverse knowledge source) |
| Integrate agents with Power Apps | Phase 5 (embedding + channel config) |
| Trigger flows from Copilot Studio agents | Phase 4 (escalation + intake flows) |
| Use adaptive cards in agent responses | Phase 4 (status + deliverables topics) |

---

## 13. Out of Scope (v1.0)

The following are explicitly excluded from the initial build to keep scope manageable before the June 2026 exam date. They may be added post-certification as portfolio extensions:

- External user authentication (Azure AD B2C / guest access)
- Power BI embedded reporting
- Custom connectors or Azure integrations
- Mobile-specific canvas app variant
- AI Builder document processing

---

## 14. Success Criteria

The project is considered complete when:

1. A client can submit an engagement request via the Canvas App or Copilot agent and see it appear in the Agent Hub
2. An agent can triage, update, and close an engagement entirely within the Model-Driven App
3. At least three Power Automate flows fire correctly in an end-to-end test scenario
4. Arcova Assist correctly handles all six defined topics with accurate Dataverse responses
5. The GitHub repository is fully documented and suitable for sharing with hiring managers or certification evaluators
6. The build covers all AB-410 exam skill domains listed in Section 12

---

*This document will be version-controlled in the project GitHub repository. Updates will be logged in the Document Control table at the top of this file.*
