# Personas & User Stories
### Arcova Engage | Power Platform Portfolio Project

---

## Purpose

This document defines the user roles the Arcova Engage solution serves and the user stories each role's needs translate into. It maps every major capability built across Phases 0–4 back to the role it exists to serve, so that the solution reads as requirements-driven rather than feature-driven.

> **A note on timing.** These user stories were formalized during Phase 5 documentation to articulate, in standard role / goal / benefit form, the requirements the solution was built to satisfy. The capabilities themselves were built across Phases 0–4; this document makes the implicit requirements explicit.

Arcova Advisory is a fictional mid-size consulting firm. Its engagement management solution serves two primary roles -the **Agent** (internal staff) and the **Requester** (client contact) -plus one specialized sub-role, the **Manager / Approver**. The personas below are role-based, with named examples drawn from the solution's actual seeded data.

---

## Role 1 -The Engagement Agent

**Surface:** Arcova Engage Agent Hub (model-driven app)
**Identity type:** Dataverse `User` record, holding the custom **Arcova Agent** security role
**Embodied by:** John Rolzhausen (primary), Cindy Rolzhausen

The Agent is internal Arcova staff who triages incoming client requests, manages engagements through their lifecycle, owns deliverables and communications, and resolves or escalates issues. Agents work primarily in the model-driven Agent Hub, where dashboards surface their workload and SLA pressure, and where the engagement lifecycle is tracked through a guided Business Process Flow.

Agents are assigned engagements automatically by category. Per the Phase 3 routing configuration, **John** is the default agent for Advisory, Technology, and Other categories; **Cindy** handles Compliance and Transformation.

### User stories

- *As an agent, I want to see the engagements assigned to me, sorted by SLA pressure, so that I can work the most urgent items first.* > My Workload dashboard; My Active Engagements view.
- *As an agent, I want to see which engagements are unassigned or breaching SLA, so that nothing falls through the cracks.* > SLA & Triage Health dashboard; Unassigned and Breaching SLA views.
- *As an agent, I want new engagements routed to the right person automatically, so that triage isn't a manual bottleneck.* > Phase 3 Intake Router (category > default agent).
- *As an agent, I want to be notified when an engagement breaches its SLA, so that I can act before the client escalates.* > Phase 3 SLA Breach Alert (scheduled flow, Teams + email).
- *As an agent, I want to escalate an engagement and have the reason captured, so that the audit trail reflects why.* > Escalation business rule; escalation flow with audit Communication.
- *As an agent, I want to be guided through the engagement lifecycle, so that I follow a consistent process from intake to closure.* > Engagement Lifecycle Business Process Flow.

---

## Role 2 -The Client Requester

**Surface:** Arcova Engage Client Portal (canvas app) + the embedded Arcova Assist agent
**Identity type:** Dataverse `Contact` record, linked to a parent `Account`, holding the custom **Arcova Requester** security role
**Embodied by:** Jordan Mills (primary example), and the other seeded client contacts

The Requester is a client-side contact at one of Arcova's client organizations. They submit new engagement requests, track the progress of their existing engagements, review deliverables, raise urgent issues, and ask process questions -all through the self-service Client Portal and the conversational Arcova Assist agent embedded within it. Requesters never touch the internal Agent Hub; their visibility is scoped to their own organization's records.

The richest example is **Jordan Mills**, Operations Director at **Northpoint Logistics** -Arcova's most active client account. Other seeded requesters populate additional client organizations: Aisha Patel and David Kowalski (Meridian Health Partners), Lina Okafor and Thomas Reyes (Verity Financial Group), Rebecca Tanaka (Cascadia Manufacturing), Sarah Whitfield (Brightway Retail Co), and Marcus Chen (Northpoint Logistics).

### User stories

- *As a requester, I want to submit a new engagement request through a simple form, so that I can get help without emailing my account contact.* > Client Portal Submit screen; Arcova Assist Submit topic.
- *As a requester, I want to see the status of my engagements at a glance, so that I know where things stand.* > Client Portal Engagements gallery; Arcova Assist Check Engagement Status topic.
- *As a requester, I want to ask "what's the status of my request?" in plain language and get an instant answer, so that I don't have to navigate menus.* > Arcova Assist conversational interface (embedded in the portal).
- *As a requester, I want to see the deliverables tied to my engagements, so that I can track what's coming and when.* > Client Portal sub-galleries; Arcova Assist List My Deliverables topic.
- *As a requester, I want to add a note to an engagement, so that I can give my agent additional context.* > Client Portal Add Note; Communication records.
- *As a requester, I want to flag an urgent issue and have the right person notified immediately, so that critical problems get fast attention.* > Arcova Assist Escalate an Issue topic > Phase 3 escalation flow.
- *As a requester, I want answers to common process and SLA questions without waiting for a person, so that I'm not blocked on simple things.* > Arcova Assist FAQ knowledge source (`arc_faq`).
- *As a requester, I want to only ever see my own organization's information, so that my data stays private.* > Account-scoped security across the portal and the agent.

---

## Sub-role -The Manager / Approver

**Surface:** Microsoft Teams (Approvals), invoked from the Agent Hub workflow
**Identity type:** A senior `User` (an Agent acting in an approving capacity)
**Embodied by:** Cindy Rolzhausen

The Manager is a senior agent who approves engagement closures and acknowledges escalations. This is not a separate person-type so much as an elevated responsibility layered onto the agent role -the manager email is externalized via the `arc_ManagerEmail` environment variable so it can be reconfigured per environment without code changes.

### User stories

- *As a manager, I want to approve or reject an engagement closure before it's finalized, so that closures get a second set of eyes.* > Phase 3 Closure Approval flow (Teams Approvals).
- *As a manager, I want to be alerted to escalations and SLA breaches, so that I have visibility into at-risk work.* > Phase 3 escalation and SLA breach flows (CC manager).

---

## Test Personas & Developer Identities

A handful of identities in the seeded data exist specifically to support development and testing. They are documented here for transparency, so that a reviewer examining the data understands their purpose.

- **Simon Hamilton** (CEO, Test Account 2) -a deliberate **single-engagement** test persona, created to verify the "one engagement, no disambiguation" path in the Arcova Assist Check Engagement Status topic.
- **Alice Katz** (CFO, Test Account) -a deliberate **zero-engagement** test persona, created to verify the empty-state handling when a requester has no active engagements.
- **John Rolzhausen** and **Cindy Rolzhausen** -exist in two capacities each. As `User` records they are the Arcova **agents** (and Cindy, the approval manager). As `Contact` records they also serve as **test requesters** linked to a seeded account (John at Northpoint Logistics; Cindy as the "Test Requester" contact at TEST - Acme Corp). This dual existence is deliberate: it lets the developer test identity-scoped requester features (which resolve the authenticated user to a Contact and Account) without standing up separate Microsoft 365 accounts for fictional client personnel. In a production deployment, agents and requesters would be distinct individuals.
- **Test Requester** / **TEST - Acme Corp** -scaffolding records used during flow development (the TEST-prefixed convention from Phase 3), kept isolated from the realistic seeded data.

> **Why this is documented rather than cleaned up.** These identities serve a real testing purpose and are referenced throughout the Phase 1–4 test scenarios and screenshots. Renaming or reassigning them at the documentation stage would invalidate already-verified tests and captured evidence for purely cosmetic gain. Documenting their purpose plainly is the more honest -and more maintainable -choice.

---

## Role-to-Surface Summary

| Role | Primary surface | Identity type | Security role |
|---|---|---|---|
| Engagement Agent | Agent Hub (model-driven) | User | Arcova Agent |
| Client Requester | Client Portal (canvas) + Arcova Assist | Contact > Account | Arcova Requester |
| Manager / Approver | Teams Approvals (from Agent Hub) | Senior User | Arcova Agent |

The two primary roles map cleanly onto the two front-end surfaces, which share the single Dataverse data model beneath them. This separation -staff on the model-driven app, clients on the canvas app and conversational agent, both on one schema -is the architectural backbone of the solution.
