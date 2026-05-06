# Phase 2: Client Portal
### Arcova Engage | Power Platform Portfolio Project

---

## Phase Overview

| Field | Detail |
|---|---|
| **Phase** | 2 - Client Portal |
| **Status** | Complete - 5/6/2026 |
| **Goal** | Build the canvas application that requesters use to submit, track, and discuss their engagements - including security, identity resolution, a design system, navigation, full CRUD against Dataverse, and a polished UI ready for the Phase 4 Copilot embed |

This phase delivered the requester-facing surface of the Arcova Engage solution. Where Phase 1 built the staff-facing Agent Hub on the model-driven app foundation, Phase 2 builds the client-facing Client Portal on the canvas app foundation. Both surfaces share the Phase 0 Dataverse schema; together they form a two-sided application.

---

## Step 1 - Security Model & Identity Resolution

### 1.1 Custom Security Role

Created a custom security role named **Arcova Requester** by copying the **Basic User** role as a baseline.

| Field | Value |
|---|---|
| **Display name** | Arcova Requester |
| **Applies to** | All Users |
| **Description** | Security role for Arcova Advisory client portal users. Grants read access to their own engagement, deliverable, and communication records, and create access for new engagements. Read-only access to account and contact data. No administrative privileges. |

### 1.2 Custom Table Privileges

Org-level read access on operational tables paired with strict app-level filtering creates defense in depth. SLA Configuration is excluded entirely - requesters have no business case to read SLA rules.

| Table | Create | Read | Write | Delete | Append | Append To |
|---|---|---|---|---|---|---|
| arc_engagement | Org | Org | None | None | None | Org |
| arc_deliverable | None | Org | None | None | None | None |
| arc_communication | Org | Org | None | None | Org | Org |
| arc_slaconfig | None | None | None | None | None | None |

> **Why org-level read instead of user-level read:** Engagements are owned by the assigned agent or the system, not the requester. User-level read would return zero records. Org-level read combined with strict app-level filtering at every gallery and lookup is the correct compensating control for this scenario.

### 1.3 Standard Table Privileges

Read-only on Account and Contact - the portal needs to resolve the requester's contact record and display their account name, but never modifies client data.

| Table | Create | Read | Write | Delete |
|---|---|---|---|---|
| Account | None | Org | None | None |
| Contact | None | Org | None | None |

### 1.4 Identity Resolution Architecture

The canvas app resolves the logged-in Microsoft 365 user to a Dataverse Contact record, then derives the parent Account from that Contact. The resolved variables (`varUserContact` and `varUserAccount`) become the security scope used by every gallery filter, every lookup, and every Patch operation downstream.

**Production resolution path:**

```powerfx
LookUp(Contacts, Email = User().Email) → varUserContact
AsType(varUserContact.'Company Name', Accounts) → varUserAccount
```
**Dev mode persona switcher:**

Because the seeded test contacts do not have matching M365 accounts, a dev-mode persona switcher provides the same effect manually. A single boolean variable (`varDevMode`) gates whether the switcher is shown. When `varDevMode = false` (production), the switcher is hidden and identity resolves automatically from the logged-in user. When `varDevMode = true` (dev/portfolio), the switcher displays a contact picker that populates the same variables manually. The rest of the app behaves identically in both modes - it cannot tell whether the variables came from email resolution or manual selection.

This architecture decouples the security boundary from the development workflow. The production code path is real and correct; the dev workflow doesn't compromise the security model.

---

## Step 2 - Design System, App Shell & Components

### 2.1 Canvas App Configuration

| Field | Value |
|---|---|
| **App Name** | Arcova Engage Client Portal |
| **App Layout** | Responsive |
| **Orientation** | Landscape |
| **Modern Controls and Themes** | Enabled |

> **Responsive vs. Fixed:** Power Apps consolidated the legacy "Scale to Fit / Lock Aspect Ratio / Lock Orientation" toggles into a single `App Layout` dropdown. Responsive replaces the previous "Scale to Fit OFF" pattern - the screen tracks `App.Width` and `App.Height` dynamically, which is what the container-based layout requires.

### 2.2 Design Tokens

Defined as named formulas in `App.Formulas`. Named formulas are author-time constants - they update instantly across the app when changed, unlike `Set()` variables which require re-running OnStart.

The token system covers:

- **Colors**: 10 brand colors (primary, accent, surface, card, border, text variants) plus 6 status colors and 4 priority colors mapped to the Phase 0 option set members
- **Typography**: One font (Open Sans) and four sizes (Lg/Md/Sm/Xs)
- **Spacing**: Three scale steps (Sm/Md/Lg)
- **Shape**: Three radius steps (Sm/Md/Lg)
- **Layout**: Fixed left nav width (220) and header height (64)

Every visible color, font, padding, and border in the app references a token - no hardcoded values. This means a single token update propagates instantly to every control that references it.

### 2.3 App Shell


A container-based layout pattern that all five screens share:

```
Screen
└── conAppShell (Vertical, fills screen)
├── conNavBar (Horizontal, full width, fixed 64px height)
└── conBody (Horizontal, fills remaining height)
├── conLeftNav (Vertical, fixed 220px width)
└── conContent (Vertical, fills remaining width)
```

Each screen renames the shell containers with a screen-specific suffix (e.g., `conAppShell_Home`, `conAppShell_Submit`) for disambiguation. A `scrTemplate` screen with the empty shell exists as the duplication source for new screens.

> **Why suffix the containers:** Modern Power Apps allows duplicate control names across screens, but disambiguation makes Tree view scanning, formula bar references, and debugging significantly easier. `conAppShell_Home` reads more clearly than `conAppShell_1` in any debugging context.

### 2.4 Data Sources

Five Dataverse tables connected via the modern Microsoft Dataverse connector:

| Table | Purpose |
|---|---|
| Contacts | Identity resolution; populating dropdowns and lookups |
| Accounts | Account scope for security filters; display values |
| Engagements | Submission target; gallery source |
| Deliverables | Sub-gallery on detail screen |
| Communications | Sub-gallery on detail screen; new note destination |

`arc_slaconfig` is deliberately excluded - the requester has no security access to it (per Section 1.2) and the portal has no need to read SLA configuration directly. SLA logic is server-side responsibility (Phase 3 intake flow).

### 2.5 Components Strategy Decision

The original plan used a **Component Library** with a reusable `cmpStatusBadge`. The library was created and the badge built with a `StatusValue` input property. However, two issues surfaced:

1. **Design tokens defined in the consuming app are not visible inside the component library.** Tokens like `radSm` and `clrStatusActive` could not be referenced from inside the library - the library is a separate canvas environment with its own scope.
2. **Components cannot be used inside Gallery templates** (error PA2122). The status badge needed to render inside the engagement gallery row, but Power Apps explicitly disallows this.

The decision was to use **in-app components** (built in the canvas app's own Components view, not the separate library) for screens, and **inline patterns** for galleries. The status badge component still earns its keep on screens like the engagement detail summary card, but galleries replicate the badge logic inline using the same Switch pattern. This decision is explicitly documented in Lessons Learned.

### 2.6 App.OnStart Foundation

Initial variable declarations for identity, dev mode, and submit-screen state. The full identity resolution logic lives here, gated by `varDevMode`. In production mode it auto-resolves; in dev mode it leaves the variables blank for the persona switcher to populate.

---

## Step 3 - Home Screen

The home screen confirms identity, gives the requester a snapshot of their activity, and routes them to the two primary actions. It is the first screen that exercises the variables and components built in Step 2.

### 3.1 Persona Switcher

The persona switcher renders only when both `varDevMode = true` and `varUserContact` is blank. Once a persona is selected, the variables populate and the switcher disappears for the rest of the session.

| Component | Purpose |
|---|---|
| Title and subtitle labels | Communicate that this is a dev-only feature |
| Combo box bound to Contacts | Search-as-you-type contact picker |
| Continue button (disabled until selection) | Resolves the chosen contact and account into the session variables |

The Continue button's OnSelect runs three operations: a fresh LookUp on the Contact GUID (because the combo box's `.Selected` returns only a partial record), an `AsType()` against the polymorphic `Company Name` field to derive the Account, and a final state set to clear the loading flag.

### 3.2 Welcome Header

A two-label hierarchy:
- **Heading**: `"Welcome, " & varUserContact.'First Name'` at 24pt semibold
- **Subtitle**: `varUserAccount.'Account Name'` at fntSizeMd in muted text

Two separate labels rather than a single concatenated string, because heading and context need different visual weights.

### 3.3 Activity Snapshot Cards

Three cards showing counts of the requester's engagements by status group. Each card has a large count number above a descriptive caption.

| Card | Status Group |
|---|---|
| Active Engagements | Active, On Hold, Escalated |
| Awaiting Triage | New, In Triage |
| Closed in Last 7 Days | Closed (with `'Closed Date' >= DateAdd(Today(), -7, TimeUnit.Days)`) |

The count formulas use the canonical Dataverse choice column comparison pattern: `'Status (arc_status)' = 'Engagement Status'.'Active'` (named option set member, not `.Value` extraction). This pattern is delegation-safe and the only reliable way to compare choice columns across tenant configurations.

### 3.4 Primary Action CTAs

Two Modern Buttons in a horizontal row:

| Button | Style | Purpose |
|---|---|---|
| Submit a New Request | Filled accent (clrAccent) - primary action | Forward navigation to scrSubmit |
| View My Engagements | Outlined primary - secondary action | Forward navigation to scrEngagements |

The filled-vs-outlined treatment is the standard primary/secondary CTA pattern. The eye is drawn to "Submit" first because that's the highest-value action a requester takes; "View" is important but visually subordinate.

### 3.5 Top Navigation Bar

The "Arcova Engage" title label populates `conNavBar_Home`, anchoring brand identity at the top of the screen. The same pattern carries to all other screens via the suffix convention.

---

## Step 4 - Submit Engagement Screen

The most consequential screen for data integrity. Every record submitted here flows into the Agent Hub and triggers downstream agent action. The screen separates user-input fields from system-set fields, exposes only what a requester would meaningfully fill in, and validates submission rigorously before committing the Patch.

### 4.1 Field Selection Strategy

The `arc_engagement` table has 13 custom columns plus the primary. A requester fills in five; the rest are auto-set, agent-set, or flow-set.

| Column | Source | Notes |
|---|---|---|
| Engagement Title | Requester | Required, min 5 chars |
| Description | Requester | Required, min 20 chars |
| Category | Requester | Required choice |
| Priority | Requester | Required choice |
| Requester Notes | Requester | Optional |
| Client Organisation | Auto from `varUserAccount` | Already known - don't ask |
| Requester | Auto from `varUserContact` | Already known - don't ask |
| Submitted Date | `Now()` at Patch time | Until Phase 3 flow auto-populates |
| Status | Hardcoded "New" | Always starts as New |
| Escalated | Hardcoded "No" | Default; agent flips this if needed |
| Assigned Agent | (Phase 3 flow or manual triage) | **Deliberately excluded from Patch** |
| SLA Due Date | (Phase 3 flow) | Calculated from arc_slaconfig |
| Closed Date | (Agent action) | Set when agent closes |
| Escalation Reason | (Agent action) | Set when escalating |

> **Why Assigned Agent is excluded from the Patch:** The "Unassigned Engagements" view in the Agent Hub depends on this field being blank for newly submitted records. If the canvas app set it on creation, the triage workflow would break. This is a deliberate separation of concerns - the Client Portal owns creation; agents and Phase 3 flows own assignment.

### 4.2 Three-Zone Layout

A pattern reused on every form-style screen in the app:

```text
conSubmitForm
├── conFormHeader (fixed - back link + title)
├── conFormBody (flexible - scrollable, contains all fields)
│   ├── conSubmitFeedback (success/error banner)
│   ├── conContextRow (read-only requester/account display)
│   ├── conFieldTitle, conFieldDescription, conFieldRow, conFieldNotes
└── conActionBar (fixed - Cancel + Submit)
```
The middle zone (`conFormBody`) has `Vertical overflow = Scroll`, which makes it scroll independently while the header and action bar remain anchored. This prevents two layout-shift problems: feedback banners pushing form content down, and Submit/Cancel buttons getting buried below the fold on long forms.

### 4.3 Real-Time Validation

The Submit button's `Disabled` property evaluates the entire form state in one expression. The button stays grey until all four conditions are true: title is at least 5 chars, description is at least 20 chars, category is selected, priority is selected, and `varSubmitting` is false (preventing double-submission during a Patch in progress).

This is more elegant than letting the user click Submit and showing an error - the button visually activates the moment the form is valid, providing continuous feedback as the user fills it out.

### 4.4 The Patch Operation

```powerfx
Patch(
Engagements,
Defaults(Engagements),
{
'Engagement Title': txtTitle.Text,
'Description': txtDescription.Text,
'Category': cboCategory.Selected,
'Priority': cboPriority.Selected,
'Requester Notes': txtNotes.Text,
'Status (arc_status)': 'Engagement Status'.'New',
'Escalated': 'Escalated (Engagements)'.'No',
'Submitted Date': Now(),
'Client Organisation': varUserAccount,
'Requester': varUserContact
}
)
```

`Defaults(Engagements)` provides a base record with system defaults, then specific columns are overridden. Choice columns use bare `.Selected` (not `.Value`, not record literals); polymorphic-resolved variables (`varUserContact`, `varUserAccount`) pass directly into lookup columns.

### 4.5 Feedback Pattern

A success or error banner appears at the top of the scrollable form body (not above it). Green tint with semantic green text on success; red tint with semantic red text on error. Visible only when `varSubmitSuccess` is true or `varSubmitError` is non-empty - hidden otherwise so the form doesn't permanently carry status from prior submissions.

The error message is gated by `varDevMode` - in dev, it shows the actual `Errors()` text from Dataverse; in production, a friendly generic message. This pattern provides debugging visibility during development without exposing technical detail in deployed apps.

### 4.6 OnVisible Reset

When the screen becomes visible, `varSubmitSuccess` and `varSubmitError` reset to their starting state. This prevents stale banners from a previous submission appearing when the user returns to the screen.

---

## Step 5 - My Engagements Gallery

A delegation-safe, filterable, searchable gallery. This screen brings together several patterns that recur throughout production Power Apps work: delegation against Dataverse, choice column comparisons, gallery + filter interaction, click-to-detail navigation, and context-aware empty state UX.

### 5.1 Three-Zone Layout

The same pattern as Submit:

```text
conEngagementsForm
├── conPageHeader (fixed - back link + page title)
├── conFilterBar (fixed - search input + status chips)
└── conGalleryWrap (flexible - gallery + empty state)
```

### 5.2 Filter Bar

A horizontal row containing:
- A search text input (320px wide, prefix-matching only)
- Four status chip buttons (All, Active, Triage, Closed) representing mutually exclusive filter states

The chips use a single `varStatusFilter` string variable for selection state, set on each chip's OnSelect. Each chip's Fill and font weight are conditional on `varStatusFilter` matching its own value.

> **Why a single string variable instead of four booleans:** Mutually exclusive states are cleaner with one variable than four. Adding a fifth filter is one new chip and one new branch in the gallery filter, not a coordination headache across multiple state flags.

### 5.3 Gallery Items Formula

```powerfx
SortByColumns(
Filter(
Engagements,
'Client Organisation'.Account = varUserAccount.Account
And (
varStatusFilter = "All"
Or (varStatusFilter = "Active" And ('Status (arc_status)' = 'Engagement Status'.'Active' Or 'Status (arc_status)' = 'Engagement Status'.'On Hold' Or 'Status (arc_status)' = 'Engagement Status'.'Escalated'))
Or (varStatusFilter = "Triage" And ('Status (arc_status)' = 'Engagement Status'.'New' Or 'Status (arc_status)' = 'Engagement Status'.'In Triage'))
Or (varStatusFilter = "Closed" And 'Status (arc_status)' = 'Engagement Status'.'Closed')
)
And (
IsBlank(txtSearch.Text)
Or StartsWith('Engagement Title', txtSearch.Text)
Or StartsWith(Description, txtSearch.Text)
)
),
"arc_submitteddate",
SortOrder.Descending
)
```

Three layered filters:

1. **Account scope** (security boundary) - the persistent filter that cannot be toggled off
2. **Status filter** (chip-controlled) - drives the gallery view
3. **Search** (text input) - real-time filter as the user types

### 5.4 Delegation Considerations

`StartsWith` is used for search rather than `in` because `in` is **not delegable** in Dataverse. The trade-off is real: search is prefix-match only ("Q3" finds "Q3 advisory engagement scoping" but "scoping" does not). True full-text search requires Dataverse's quick search or Azure Cognitive Search integration, both out of scope for v1.0.

A delegation warning may appear despite the formula being delegation-safe at the operator level - the combination of multiple `Or` clauses sometimes triggers warnings even when the underlying operations delegate. With the test dataset of 20 engagements this is academic; with production data of 50,000+ records, the architecture would still scale because the operations themselves do delegate.

### 5.5 Gallery Row Template

Each row is a Modern Button styled as a card, with a transparent layered container holding the visual content. This pattern (button beneath, transparent layer above with `DisplayMode = View`) is the canonical solution to Modern Container click suppression - more on that in Lessons Learned.

The row contains three visual zones:
- **Title block**: engagement title + description, flexible width
- **Status badge**: inline pattern using Switch on the choice column
- **Date block**: "Submitted" caption + formatted date, fixed width

### 5.6 Click-to-Detail Pattern

The Gallery's OnSelect:

```powerfx
Set(varSelectedEngagement, ThisItem);
Navigate(scrEngagementDetail, ScreenTransition.Cover)
```

Storing the selected record in a global variable (rather than relying on `gallery.Selected`) preserves the selection across screen transitions. This is the canonical click-to-detail pattern in canvas apps.

### 5.7 Empty State - Four Branches

The empty state label communicates *why* the gallery is empty, with branch logic that distinguishes four scenarios:

1. **No engagements ever** (no search, "All" selected) → "You haven't submitted any engagements yet..."
2. **Combined filter empty** (search active AND non-All chip) → "No [status] engagements match your search for '[term]'. Try clearing your search or selecting 'All'."
3. **Search-only empty** (search active, "All" chip) → "No engagements match your search for '[term]'..."
4. **Filter-only empty** (no search, non-All chip) → "No engagements match the selected filter..."

The combined-filter branch is essential when filters compound - users need to know *which* filters are active so they can decide what to clear. Generic "no results" messages are a UX failure when multiple filters can independently produce empty results.

---

## Step 6 - Engagement Detail Screen

Where the click on a gallery row leads. The detail screen is read-only by design - requesters cannot modify their submission after creation. Instead, follow-up context is added through new communication records, preserving the audit trail of every interaction.

### 6.1 Three-Zone Layout

The same pattern as Submit and Engagements:

```text
conDetailForm
├── conPageHeader_Detail (fixed - back link + engagement title)
├── conDetailBody (flexible - scrollable, contains all reading zones)
└── conDetailActions (fixed - Add Note + Ask Arcova Assist)
```

### 6.2 Engagement Summary Card

A consolidated view of the engagement's metadata, organised top to bottom:

| Zone | Contents |
|---|---|
| Status row | Status badge, priority badge, conditional escalation indicator |
| Description | Caption + full description text |
| Metadata grid | Three columns: Category, Submitted Date, Assigned Agent |
| Notes (conditional) | Original requester notes from submission, only rendered if present |

The status and priority badges use the same inline Switch pattern as the gallery rows - replicating the cmpStatusBadge component's logic directly because Power Apps does not allow components inside galleries (and we want the same visual on the gallery row template).

The Assigned Agent value uses an `If()` to handle the unassigned case: `If(IsBlank(varSelectedEngagement.'Assigned Agent'), "Awaiting assignment", varSelectedEngagement.'Assigned Agent'.'Full Name')`. Newly submitted engagements have no agent yet, and "Awaiting assignment" gives the requester a meaningful read on the state rather than a confusing blank field.

### 6.3 Sub-Galleries

Two galleries on the same screen, both filtered to records linked to the parent engagement:

**Deliverables sub-gallery** - filtered by `Engagement.Engagement = varSelectedEngagement.Engagement`, sorted by due date ascending. Each row shows the deliverable name, type, status badge (with deliverable-specific Switch), and due date.

**Communications log** - filtered to the same parent engagement, sorted by Time Stamp descending so the newest interaction appears at the top. Each row shows direction and channel, the subject line, and the first 18 pixels of content.

Both galleries display empty-state messaging when no records exist for the parent engagement. This is meaningful for newly submitted engagements that have no deliverables or communications yet.

### 6.4 Add Note Modal

The "Add a Note" button on the action bar opens a modal dialog over the entire screen. The modal is implemented as a screen-level container (not nested inside `conContent_Detail`, because vertical containers cannot overlap their siblings - they stack them).

```text
scrEngagementDetail
├── conAppShell_Detail (the entire app shell)
└── conNoteDialog (sits ON TOP at screen level, fills the screen)
└── conNoteDialogCard (centered fixed-size card)
```

The dialog's container fills the screen with a semi-transparent black backdrop (`RGBA(0, 0, 0, 0.5)`), creating the "screen behind visible but locked" effect. The card itself has fixed dimensions (480x320) and centers via the parent's Justify and Align settings.

The Add Note button's OnSelect Patches a new Communication record with `Direction = Inbound`, `Channel = Portal`, and `Time Stamp = Now()`. The Time Stamp field is required on the Communications table - until the Phase 3 auto-population flow exists, every client creating a Communication record must populate Time Stamp explicitly.

After the Patch succeeds, `Refresh(Communications)` updates the local cache so the new note appears immediately in the gallery on the same screen.

### 6.5 Action Bar

Two buttons at the bottom of the screen:

| Button | Style | Behavior |
|---|---|---|
| Add a Note | Outlined primary | Opens the modal dialog |
| Ask Arcova Assist | Filled accent | Navigates to scrChat (Phase 4 will activate the Copilot here) |

Both buttons disable when the engagement's status is Closed - closed records are read-only and shouldn't accept new communications or queries through the AI assistant.

---

## Step 7 - Chat Screen Placeholder

A deliberately minimal screen built as a destination for the "Ask Arcova Assist" action. Phase 4 will replace the placeholder content with the embedded Copilot Studio agent, but the screen, the navigation hookups, and the visual structure are all in place now.

The placeholder communicates intent rather than absence:

| Element | Purpose |
|---|---|
| Centered card with icon | Visual anchor that signals "this is a real feature" |
| "Arcova Assist" title | Identifies the upcoming agent |
| "Available with the next platform update" | Professional product communication, not a TODO marker |
| Description paragraph | Sets expectations about what the Copilot will do |
| Capability list (4 items) | Previews the agent's planned topics |
| Permanently disabled CTA | Signals "intentionally inactive" pattern |

The disabled CTA pattern matters because users instinctively recognise it as "not yet active" rather than "missing functionality" or "broken." A blank screen or generic "Coming Soon" banner would suggest an unfinished build; a thoughtfully styled disabled state suggests deliberate design.

The back button uses `Back()` rather than hardcoded navigation. The chat screen has multiple entry points (the action bar on the detail screen, and the left navigation from any screen), and `Back()` returns the user to wherever they came from.

---

## Step 8 - Persistent Navigation & Polish

The section that transformed five disconnected screens into a cohesive application. The work was structured around two themes: persistent left navigation that connects every screen, and final polish across error handling, loading states, and visual consistency.

### 8.1 Left Navigation as Component

Built as a reusable component (`cmpLeftNav`) following the pattern from Matthew Devaney's [Power Apps Navigation Menu Component](https://www.matthewdevaney.com/power-apps-navigation-menu-component/) guide, customized for this project's design tokens and screen set.

The earlier decision (Step 2.5) was to avoid the component library because of design token scope issues and gallery template restrictions. By Step 8, the calculus had changed: navigation appears on every screen, has significant duplication risk, and has no relationship to galleries. This is exactly when components earn their cost. The single source of truth means one update to nav structure or styling propagates to all five screens automatically.

Input properties into the component carry the design token values that the component cannot reference directly (the design token scope problem from Step 2.5). This pattern - component as reusable structure, design tokens as configuration - is the right architecture for cross-screen UI.

### 8.2 Navigation Entries

Four entries, ordered by frequency of use:

| Entry | Modern Icon | Destination |
|---|---|---|
| Home | Home | scrHome |
| Submit | Add | scrSubmit |
| Engagements | BulletedList | scrEngagements |
| Chat | Chat | scrChat |

The icons are Modern (Fluent-style) icons from the Insert > Display > Icon control. Modern Icons use string identifiers (e.g., `"Home"`) rather than the legacy `Icon.Home` enum - the two systems cannot be mixed in the same property reference.

### 8.3 Active State

Each screen sets a `varCurrentScreen` string in its OnVisible:

| Screen | varCurrentScreen value |
|---|---|
| scrHome | "Home" |
| scrSubmit | "Submit" |
| scrEngagements | "Engagements" |
| scrEngagementDetail | "Engagements" |
| scrChat | "Chat" |

The component uses this variable to style the active entry differently - lighter fill (`clrPrimaryHover`) and semibold weight - so users always know which section of the app they're in.

> **Why scrEngagementDetail also reports as "Engagements":** The detail screen is conceptually a child of the Engagements section. When a user is viewing engagement details, they're still in "Engagements" - the parent nav entry should remain highlighted to reflect that. This is the standard parent-child navigation pattern used in Microsoft 365 apps and most professional productivity tools.

### 8.4 Screen Transitions

Different transitions communicate different navigational intent:

| Navigation | Transition | Reasoning |
|---|---|---|
| Home → Submit (CTA) | Cover | Forward action, deeper context |
| Home → Engagements (CTA) | Cover | Forward action, deeper context |
| Engagements → Detail (gallery click) | Cover | Drill-down |
| Submit → Home (Cancel/Back) | UnCover | Reverse navigation |
| Engagements → Home (Back) | UnCover | Reverse navigation |
| Detail → Engagements (Back) | UnCover | Reverse navigation |
| Detail → Chat (Ask Arcova) | Fade | Modal-style context shift |
| Left nav (any → any) | None | Top-level navigation should feel instant |

The convention - animations for context shifts, no animation for nav - keeps the persistent navigation feeling responsive rather than sluggish.

### 8.5 Loading State

The submit screen and the Add Note dialog both have brief loading indicators ("Submitting..." and similar) that appear while the Patch operation runs. These are gated by `varSubmitting` and `varAddingNote` respectively. Brief because the Patch is fast on small payloads - the user sees the indicator just long enough to confirm "yes, the system received your action."

### 8.6 Error Surfacing

The error message in the submit screen is gated by `varDevMode`:

```powerfx
Set(varSubmitError,
If(varDevMode,
"Dev: " & First(Errors(Engagements, varNewEngagement)).Message,
"We couldn't submit your engagement. Please try again."
)
)
```

In dev mode, the actual Dataverse error appears (useful for debugging during portfolio review or future development). In production, a friendly generic message protects the user from technical detail. Flipping `varDevMode` to false at the top of `App.OnStart` shifts the entire app from dev behavior to production behavior in one variable change.

---

## Lessons Learned

The build surfaced several non-obvious technical patterns - some that cost real debugging time, others that came from questioning a default approach and finding a better one. Each is documented with the symptom that surfaced it, the underlying cause, and the resolution.

### Lesson 1 - Polymorphic Dataverse fields require AsType()

**Symptom**: Direct dot notation against `varUserContact.'Company Name'` failed with "The '.' operator cannot be used on Polymorphic values."

**Root cause**: The Contact table's `'Company Name'` column (`parentcustomerid`) is a Customer lookup - a polymorphic Dataverse field type that can point to either an Account or a Contact. Power Apps refuses dot notation against polymorphic values because the type cannot be determined at author time.

**Resolution**: `AsType(varUserContact.'Company Name', Accounts)` disambiguates the type at author time, returning a fully-typed Account record that can be dotted into normally. Crucially, AsType returns the full expanded record - no follow-up `LookUp(Accounts, ...)` is needed. The polymorphic reference already contains every column; AsType just unwraps it.

**Wider applicability**: Customer lookups appear on a handful of standard Dataverse tables (Contact, Opportunity, Quote) but rarely on custom tables. When working with these tables, expect this pattern. The same `AsType()` approach also disambiguates other polymorphic fields like Owner (which can point to User or Team) and Regarding fields on Activity tables.

### Lesson 2 - Dataverse statecode collides with custom status columns in formula autocomplete

**Symptom**: A formula referencing `'Status (Engagement)'` resolved at runtime to a column with only "Active" and "Inactive" options, despite the engagement table having six business statuses defined as a custom choice column.

**Root cause**: Dataverse's built-in `statecode` field exists on every table by default, separate from any custom status column. When formula autocomplete suggests "Status" or "Status (Engagement)," it can resolve to either - and frequently resolves to the system field rather than the intended custom column.

**Resolution**: Use schema-qualified display names like `'Status (arc_status)'` to explicitly reference the custom column. The schema name disambiguates against the system field. This pattern is essential whenever a custom status column exists alongside `statecode`.

**Wider applicability**: This same collision affects any custom column whose display name overlaps with a system field. The schema-qualified form is the safest reference pattern, and worth using by default whenever autocomplete shows multiple matches for a name.

### Lesson 3 - Choice column comparisons require named option set members, not text labels

**Symptom**: A Switch formula comparing `'Status (arc_status)'` against text labels like `"Active"` returned "Invalid argument type (Text). Expecting a OptionSetValue (Engagement Status) value instead."

**Root cause**: Dataverse choice columns return OptionSetValue records, not strings. Comparing them to text labels is a type mismatch. Power Apps's behavior with `.Value` extraction is inconsistent across tenant configurations - sometimes returning the numeric option set value, sometimes the text label, depending on Modern Controls and tenant build.

**Resolution**: Use the canonical pattern: `Switch(true, choiceColumn = 'OptionSetName'.'MemberName', resultIfTrue, ...)`. Both sides of the equality are OptionSetValues, so the comparison is type-safe. The `Switch(true, ...)` idiom turns Switch into a chain of conditions evaluated in order - more readable than nested If statements when comparing one value against multiple choice options.

**Wider applicability**: This pattern applies to every choice column comparison in canvas apps - not just status fields. It's worth adopting as the default approach for any Filter, Switch, or If formula that needs to test choice column values, because it works regardless of tenant configuration.

### Lesson 4 - Patch failures often misattribute to the wrong column

**Symptom**: A Patch operation against the Engagement table failed with "The type of this argument 'arc_category' does not match the expected type 'OptionSetValue (Engagement Category)'. Found type 'Record'." After several iterations adjusting the Category column's value, the actual problem was elsewhere.

**Root cause**: Dataverse rejects the entire Patch transaction when any required column is missing or has the wrong type. Power Apps surfaces the error attributed to whatever column was last evaluated, which may not be the actual constraint violation. In this case, the missing required field was `arc_submitteddate` - but the error pointed at `arc_category`.

**Resolution**: When Patch fails with a confusing error, audit the table's required columns first. A single missing required field will fail the whole transaction, regardless of which column the error message names. For the Engagement table, the deferred Phase 1 business rule for auto-populating `arc_submitteddate` means this column must be set explicitly via the Patch (`'Submitted Date': Now()`) until the Phase 3 flow takes over. The same applies to `arc_timestamp` on the Communication table.

**Wider applicability**: This is the single most useful debugging heuristic for Dataverse Patch failures. Before assuming a type mismatch, list the required columns and verify all of them are populated. The misleading error attribution is unfortunately a Power Apps platform behavior rather than a fixable bug.

### Lesson 5 - Components cannot be used inside Gallery templates

**Symptom**: Attempting to insert `cmpStatusBadge` into a gallery row template produced error PA2122: "Control of type 'CanvasComponent' is not supported as a child (or descendant) for control type 'Gallery'."

**Root cause**: Components in Power Apps are designed as singleton instances bound to specific screen positions. Gallery templates need to render the same logic N times against per-row data, and the singleton component model conflicts with that requirement at the platform level.

**Resolution**: For repeating UI elements that need to appear in galleries, replicate the component's logic inline in the template. The cmpStatusBadge component remained useful on static screens (engagement detail summary card), but the gallery row uses an inline Switch on the row's choice column data.

**Wider applicability**: This forces a deliberate component strategy. Components are appropriate for cross-screen reusable UI that does not appear in galleries (navigation, page headers, action bars). Inline patterns are required for anything that repeats per-row in a gallery. Mixing the two strategies based on context is fine and often correct.

### Lesson 6 - Modern Containers do not expose OnSelect

**Symptom**: Adding OnSelect logic directly to gallery row templates failed because Modern Containers do not have an OnSelect property. Click-to-detail navigation could not be wired through the row's outermost container.

**Root cause**: Unlike classic Containers, Modern Containers (the auto-layout vertical and horizontal containers) do not expose click event handlers. They were designed for layout, not interaction. This is a deliberate platform choice - clicks should be handled by interactive controls (buttons, icons), not layout containers.

**Resolution**: Use a Modern Button as the outermost clickable surface in the gallery row template, with the visual content layered on top in a transparent Modern Container set to `DisplayMode = View`. The button handles the click; the layered container is non-interactive (clicks pass through). This is functionally equivalent to a clickable card and is the canonical pattern in modern web component libraries (shadcn, Material UI, etc.).

**Wider applicability**: This pattern - button beneath, transparent layer above - applies anywhere a clickable card or list item is needed in a Modern Controls app. It's also more accessible than transparent overlays because the button receives proper keyboard focus and screen reader semantics.

### Lesson 7 - System-set required fields require coordination across creation paths

**Symptom**: Two missing required fields surfaced during build (`arc_submitteddate` on Engagement, `arc_timestamp` on Communication). Both were marked Required but had no UI input from the user.

**Root cause**: These fields were originally planned to be auto-populated by Power Automate flows in Phase 3, with `Now()` or `utcNow()` setting them on row creation. Until those flows exist, every client that creates records of these tables must populate the fields explicitly. The Canvas App is one such client; Phase 4's Copilot agent will be another.

**Resolution**: The Canvas App Patch operations explicitly set these fields with `Now()`. When the Phase 3 flows exist, this can either remain (defensive) or be removed (cleaner, single source of truth). The cleaner option is preferred but requires coordinated change across all clients.

**Wider applicability**: This is a pattern to recognise across any solution where business rules or flows are deferred. Required fields without UI need to be populated by *someone*, and the solution architecture must specify which surface owns the responsibility. In a multi-surface solution like this one (canvas app + model-driven app + Copilot agent + flows), each system-set required field needs an explicit owner.

### Lesson 8 - Modal overlays must be placed at the screen level, not inside layout containers

**Symptom**: Adding the Add Note modal as a sibling inside `conContent_Detail` would have caused it to push the form down rather than overlay it. The modal needs to float over the screen, not stack with siblings.

**Root cause**: Layout containers (Vertical, Horizontal) position their children according to layout rules - they cannot overlap a sibling. Vertical containers stack children top to bottom; horizontal containers place them side by side. Neither allows a child to occupy the same space as another child.

**Resolution**: The modal container is placed as a direct child of the Screen itself, not inside any layout container. At the screen level, controls are positioned by absolute X/Y/Width/Height and can overlap freely. The modal fills the entire screen via `X=0, Y=0, Width=Parent.Width, Height=Parent.Height`, and a semi-transparent fill creates the "screen behind visible but locked" backdrop effect.

**Wider applicability**: This is the only way to create modal overlays in Power Apps with auto-layout containers. It applies to any floating UI element - dialog boxes, dropdown menus, tooltip-style popovers, full-screen loading states.

### Lesson 9 - Modern Icons use string identifiers, not the Icon enum

**Symptom**: An app navigation collection that worked with classic icons (`Icon: Icon.Home`) failed when attempting to use Modern Icon names (`Icon: Icon.Chat`). The Icon enum did not have entries for many Modern icons.

**Root cause**: Power Apps has two icon systems. Classic icons use an enum (`Icon.Home`, `Icon.Add`, `Icon.Person`, etc.) with a defined set of values. Modern Icons - the Fluent-style icons available through Insert > Display > Icon - use string identifiers (`"Home"`, `"Add"`, `"BulletedList"`, etc.). The two cannot be mixed in the same property reference.

**Resolution**: Use string identifiers throughout when working with Modern Icons. Collections driving icon-based UI should store strings (`Icon: "Home"`) rather than enum references. Inside components or controls, the Modern Icon control's `Icon` property reads the string and resolves it to the actual icon glyph from the Fluent catalog.

**Wider applicability**: This decision tree applies to any project starting fresh on Modern Controls. Mixing classic and Modern icon references in the same app is technically possible but creates confusion. Picking one system (Modern, for any new app) and using it consistently is the cleaner pattern.

### Lesson 10 - Combined filters need explicit empty-state messaging

**Symptom**: During UX testing, the engagement gallery's empty state didn't immediately register when a search term and a status filter combined to produce zero results. The empty state message named one filter as the cause, but not the combined state.

**Root cause**: The original empty state had three branches (no records, search-only, filter-only), but no branch for the combined-filter case. When both filters were active, the user saw a generic search-related message that didn't acknowledge the additional status filter narrowing the results.

**Resolution**: A four-branch If statement covers the four possible empty-state scenarios independently. The combined-filter branch explicitly names both active filters and offers the user multiple paths to clear the empty state. This is the canonical pattern for productivity tools that combine filters - persistent state is preferred (cleaner UX), but the empty state must communicate compound results clearly when they occur.

**Wider applicability**: Any time a screen has multiple independent filters that can each produce an empty result, the empty state needs branch logic that distinguishes which filter (or combination) is responsible. Generic "no results" messages fail when users cannot tell which filter to relax.

---

## Phase 2 Completion Criteria

### Security
- [x] Custom security role `Arcova Requester` created with least-privilege design
- [x] Role assigned to development user account
- [x] Identity resolution from User().Email to Contact and Account
- [x] Dev-mode persona switcher implemented as security-equivalent dev workflow

### Design System
- [x] Design tokens defined as App.Formulas (colors, typography, spacing, shape, layout)
- [x] No hardcoded colors, fonts, or spacing values in any control
- [x] Container-based responsive layout pattern used for app shell
- [x] scrTemplate screen exists as duplication source for new screens
- [x] Modern Controls used consistently throughout

### Screens
- [x] scrHome built with welcome header, snapshot cards, and CTAs
- [x] scrSubmit built with three-zone layout, validation, and Patch
- [x] scrEngagements built with delegation-safe filtering and search
- [x] scrEngagementDetail built with summary card and two sub-galleries
- [x] scrChat placeholder built and ready for Phase 4 Copilot embed

### Data Operations
- [x] Engagement creation Patch verified end-to-end (Canvas App → Agent Hub)
- [x] Communication (Add Note) Patch verified end-to-end
- [x] Filter formulas delegate correctly against Dataverse
- [x] Choice column comparisons use canonical named option set member pattern

### Navigation
- [x] Persistent left navigation built as reusable component
- [x] Active state styling correct on all five screens
- [x] Parent-child navigation pattern (Detail screen highlights "Engagements")
- [x] Screen transitions follow forward/reverse/modal/instant convention

### Polish
- [x] Empty states with context-aware messaging on all data-driven screens
- [x] Loading indicators on Patch operations
- [x] Error messages gated by varDevMode (technical in dev, friendly in production)
- [x] All Modern Controls use Fluent-style icons consistently

### Documentation
- [x] Screenshots captured throughout the build (~32 portfolio images committed to /screenshots/phase2/)
- [x] PHASE_2.md committed to GitHub with full architectural rationale