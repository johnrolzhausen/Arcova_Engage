# Phase 0: Foundation
### Arcova Engage | Power Platform Portfolio Project

---

## Phase Overview

| Field | Detail |
|---|---|
| **Phase** | 0 - Foundation |
| **Status** | Complete - 4/23/2026 |
| **Goal** | Establish the project infrastructure, Power Platform environment, and Dataverse schema before any app, flow, or agent is built |


## Step 1 - GitHub Repository Setup

### 1.1 Repository Details

- Repository Name: **Arcova-Engage**
- Address: github.com/johnrolzhausen/Arcova_Engage
- Added a README.md 
- Added a .gitignore with VisualStudio template
- License: MIT 

### 1.2 Create the Folder Structure

```
arcova-engage/
├── README.md
├── PROJECT_BRIEF.md                
├── docs/
│   ├── architecture/
│   │   └── .gitkeep
│   ├── requirements/
│   │   └── DATA_MODEL.md           
│   ├── personas/
│   │   └── .gitkeep
│   └── phases/
│       ├── PHASE_0.md              
│       ├── PHASE_1.md
│       ├── PHASE_2.md
│       ├── PHASE_3.md
│       ├── PHASE_4.md
│       └── PHASE_5.md
├── flows/
│   └── .gitkeep
├── copilot/
│   └── .gitkeep
└── screenshots/
    └── .gitkeep
```

---

## Step 2 - Power Platform Environment Setup

### 2.1 Environment Details

In the **Power Platform Admin Centre** (admin.powerplatform.microsoft.com):

- Environment Name: **Arcova Engage - Dev**
- Environment Type: **Developer** 
- Region: **USA**
- Dataverse Enabled: **Yes**
- Currency: **US Dollar (USD)**

---

## Step 3 - Solution Setup

### 3.1 Solution Details

Environment: **Arcova Engage - Dev** 

| Field | Value |
|---|---|
| **Display name** | Arcova Engage |
| **Name** | AarcovaEngage |
| **Publisher** | Arcova Advisory Created **NEW** (see below) |
| **Version** | 1.0.0.0 | 

### 3.2 Publisher Details

When creating the solution, you'll need to create a Publisher:

| Field | Value |
|---|---|
| **Display name** | Arcova Advisory |
| **Name** | arcova |
| **Prefix** | `arc` |

---

## Step 4 - Dataverse Schema Build

Refer to **docs/requirements/DATA_MODEL.md** for the full schema specification.

| Order | Table |
|---|---|
| 1 | Account (standard) | 
| 2 | Contact (standard) |
| 3 | arc_sla_config |
| 4 | arc_engagement |
| 5 | arc_deliverable |
| 6 | arc_communication |

---

## Phase 0 Completion Criteria

- [x] GitHub repository is public and has the full folder structure
- [x] `PROJECT_BRIEF.md` and `PHASE_0.md` are committed
- [x] `DATA_MODEL.md` is committed with the full schema
- [x] Power Platform Developer environment `Arcova Engage - Dev` is provisioned
- [x] Solution `Arcova Engage` exists with publisher prefix `arc`
- [x] All 6 Dataverse tables are built inside the solution
- [x] All relationships are configured correctly
- [x] Schema screenshot committed to GitHub

