# Arcova Engage: Solution Architecture

A high-level view of how the four surfaces, the automation layer, and the Dataverse data model relate. This diagram renders natively in GitHub.

```mermaid
flowchart TB
    %% ===== Users layer =====
    subgraph USERS [Users]
        direction LR
        AGENT([Engagement Agent])
        REQ([Client Requester])
        MGR([Manager / Approver])
    end

    %% ===== Identity layer =====
    IDENTITY{{"Microsoft Entra ID<br/>SSO + Security Roles<br/>Account-scoped access"}}

    %% ===== Surface layer =====
    subgraph SURFACES [Application Surfaces]
        direction LR
        HUB["Agent Hub<br/>(Model-driven app)"]
        subgraph PORTALGROUP [Client Portal - Canvas app]
            direction TB
            PORTAL["Client Portal<br/>screens and galleries"]
            ASSIST["Arcova Assist<br/>(Copilot Studio agent,<br/>embedded via PCF)"]
        end
    end

    %% ===== Automation layer =====
    FLOWS["Power Automate<br/>7 flows: intake routing, SLA breach,<br/>status notify, closure approval,<br/>escalation, auto-timestamps<br/>+ Copilot helper flows"]

    %% ===== Data layer =====
    DATAVERSE[("Microsoft Dataverse<br/>Account, Contact, Engagement,<br/>Deliverable, Communication,<br/>SLA Config, Category Assignment, FAQ")]

    %% ===== Connections =====
    AGENT --> HUB
    REQ --> PORTAL
    REQ --> ASSIST
    MGR -. approvals .-> FLOWS

    IDENTITY --- HUB
    IDENTITY --- PORTAL
    IDENTITY --- ASSIST

    HUB --> DATAVERSE
    PORTAL --> DATAVERSE
    ASSIST --> FLOWS
    PORTAL -. invokes .-> FLOWS

    FLOWS --> DATAVERSE
    FLOWS -. Teams + email .-> MGR

    %% ===== Styling =====
    classDef userNode fill:#e8eef7,stroke:#0f3060,stroke-width:1px,color:#0f3060;
    classDef surfaceNode fill:#0f3060,stroke:#0f3060,stroke-width:1px,color:#ffffff;
    classDef autoNode fill:#fff4e6,stroke:#B85C00,stroke-width:1px,color:#7a3d00;
    classDef dataNode fill:#eef7f0,stroke:#1e7d4f,stroke-width:1px,color:#14593a;
    classDef idNode fill:#f3eef7,stroke:#5a2d82,stroke-width:1px,color:#3d1d57;

    class AGENT,REQ,MGR userNode;
    class HUB,PORTAL,ASSIST surfaceNode;
    class FLOWS autoNode;
    class DATAVERSE dataNode;
    class IDENTITY idNode;
```

---

## How to read this diagram

**Users (top).** Three roles drive the system: the **Engagement Agent** (internal staff), the **Client Requester** (client-side contact), and the **Manager / Approver** (a senior agent who approves closures and is notified of escalations). These map to the profiles in `personas.md`.

**Application surfaces (middle).** Agents work in the model-driven **Agent Hub**. Requesters use the canvas **Client Portal**, which now hosts the **Arcova Assist** Copilot Studio agent embedded directly in its Chat screen via a PCF control. The nesting in the diagram reflects that the agent lives inside the portal rather than as a separate destination.

**Identity (cross-cutting).** Microsoft Entra ID provides single sign-on and the security model. All three surfaces resolve the signed-in user to their Dataverse identity, and account-scoping ensures requesters see only their own organization's records.

**Automation.** The seven Power Automate flows handle data hygiene, notifications, SLA enforcement, closure approvals, and escalation, plus the helper flows that shape data for the Copilot agent. The agent reaches Dataverse through these flows rather than querying directly.

**Data (bottom).** A single Dataverse data model underlies every surface. The full schema, including columns and relationships, is documented in `DATA_MODEL.md`.

The defining characteristic of the architecture is that two distinct front-end experiences, staff and client, plus a conversational agent, all sit on one shared data model with one shared automation backbone. Each layer consumes the ones beneath it.
