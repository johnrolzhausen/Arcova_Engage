# Phase 4 - Copilot Agent (Arcova Assist)

**Status:** Complete
**Build dates:** May 2026
**Solution:** Arcova Engage
**Environment:** Arcova Engage - Dev
**Publisher prefix:** `arc`

## Phase summary

Phase 4 built **Arcova Assist**, the Copilot Studio agent that lives inside the Arcova Engage solution. The agent provides a conversational interface that complements the Phase 1 Agent Hub (model-driven app for staff) and the Phase 2 Client Portal (Canvas app for clients), giving requesters a third way to interact with their engagements - one driven by natural language rather than forms and grids.

Where prior phases built data, automation, and form-based UI, Phase 4 added an intelligence layer on top of all of them. The agent reads from the Dataverse schema established in Phase 0, displays data through patterns parallel to Phase 2's Canvas App, invokes the flows built in Phase 3, and respects the security boundaries set across the entire solution. Every cross-phase touchpoint visible in this phase was deliberately designed in earlier phases to be reused here.

The phase is structured around eight sections: an architectural scoping section, agent shell and authentication setup, a knowledge source for FAQs, four authored topics (Check Engagement Status, List My Deliverables, Submit New Engagement, Escalate an Issue), and this documentation section. Each section delivered against a specific AB-410 exam skill area, with the four authored topics collectively demonstrating the full range of conversational agent capabilities: data queries with adaptive cards, multi-turn slot-filling with Dataverse writebacks, external flow invocation, and dynamic knowledge retrieval.

---

## Section 1 - Architectural Scoping

Section 1 didn't build anything technical. It locked in the architectural decisions that shaped the rest of Phase 4 before any code was written. This is consistent with prior phases' practice of scoping deliberately before building, and Phase 4's architectural decisions had more downstream consequences than any phase to date.

### Original scoping decisions (locked at start of phase)

The eight decisions taken at scoping time:

| # | Decision | Rationale at scoping |
|---|---|---|
| 1 | Generative orchestration (vs. classic) | Modern Copilot Studio's default; description-driven topic discovery without per-trigger-phrase maintenance |
| 2 | Agent contained in the Arcova Engage solution | Solution portability; agent travels with the rest of the project artifacts |
| 3 | Microsoft Entra authentication, configured immediately | SSO from the start; avoids retrofit work; production-pattern from day one |
| 4 | Direct Dataverse connector for simple reads, Power Automate for complex operations | Right tool for each operation; Section 4-5 use direct reads via flow, Section 6-7 use writebacks via flows |
| 5 | Adaptive cards authored inline in topics | Visual richness for engagement detail and submission confirmation surfaces |
| 6 | Dedicated `arc_faq` table as knowledge source | Curated content; explicit ownership of FAQ data; testable knowledge boundary |
| 7 | Test pane during build, demo website publish at end of phase | Iterative testing during build, full publish only after all topics are verified |
| 8 | Six topics: Greeting/Fallback, FAQs, Check Status, List Deliverables, Submit, Escalate | Coverage of the AB-410 exam skills with deliberate scope for v1.0 |

### How decisions evolved during build

Two of the eight original decisions encountered platform-version constraints during build and required adaptation. These are worth documenting honestly because they illustrate the gap between scoping-time assumptions and build-time reality.

**Decision 5 (adaptive cards authored inline) - evolved.** The original scoping assumed JSON-format adaptive cards with `${variable}` placeholders, the industry-standard adaptive card templating syntax. During Section 4 build, this approach hit a platform constraint: the "Ask with adaptive card" node in current Copilot Studio requires an Action.Submit button and a YAML-format output schema, and the binding mechanism for `${variable}` placeholders is not cleanly exposed through any UI surface in this version. After extensive exploration, the architecturally correct path was to use the **Formula card format** instead - a Power Fx record literal where variable references resolve at runtime through Copilot Studio's expression engine. The final cards use Formula format with direct `Global.SelectedEngagement.EngagementTitle`-style references and produce visually identical results to what the JSON approach would have produced. Section 4's Lessons Learned section documents this discovery in detail; it became the dominant Phase 4 platform finding.

**Decision 4 (direct Dataverse connector + Power Automate) - clarified.** The original scoping was correct in spirit but vague on execution. During Section 4 build, the clearer pattern emerged: **all four authored topics invoke helper flows rather than calling the Dataverse connector directly**, even for simple reads. The reasoning is platform-specific: Copilot Studio's modern flow-call trigger removes the ability to test flows manually, but it also enables much cleaner topic-to-Dataverse data shaping in the flow (where iteration, lookups, and conditional logic are easier to author than in topic Power Fx). The "use flows for complex operations" guidance evolved into "use flows for all data operations." Section 4's helper flow `[Arcova Engage] Copilot - Get User Engagements` set the pattern that Sections 5, 6, and 7 all reused (Sections 5 and 6 with their own helper flows; Section 7 reuses Section 4's flow directly).

The other six decisions held up cleanly through build with no significant deviation.

### Decisions added during build

Two architectural decisions were added during build that weren't in the original scoping but became important:

**Account-scoped data security** - All four authored topics filter their data queries by the user's account, not by the user's contact ID or any narrower scope. This matches Phase 2's Canvas App security model and provides architectural consistency: a user sees their organization's engagements and deliverables, not just their personal ones. This decision was confirmed explicitly during Section 5 scoping (Deliverables) when the question of "personal scope vs. account scope" came up, and the answer for cross-topic consistency was account scope.

**Global variable scope for cross-branch state** - During Section 4 build, the `SelectedEngagement` variable was originally created as Topic-scoped. When the topic was refactored to converge the single-engagement and multi-engagement branches at a shared display node, the Topic-scoped variable didn't reliably resolve at the convergence point. Promoting it to Global scope fixed the issue. This pattern - "variables set in multiple branches and read at a shared downstream node should be Global" - applied consistently to Section 7's escalation topic as well.

---

## Section 2 - Agent Shell, SSO Authentication, and Identity Resolution

**Purpose:** Create the Arcova Assist agent within the solution, configure Entra authentication, and build the identity resolution flow that maps the authenticated user to a Dataverse Contact and Account.

### Agent creation

The agent was created via the **solution context** (not directly in Copilot Studio's standalone interface). This ensures the agent is part of the Arcova Engage solution and travels with it for export/import scenarios.

Agent settings:
- **Name:** Arcova Assist
- **Content moderation:** High
- **Connected agents:** Disabled (no multi-agent orchestration in scope)
- **File uploads:** Disabled (no need for users to upload files via the agent)
- **Web search:** Disabled (knowledge should come from controlled sources, not the web)

These settings tighten the agent's behavior to deliberate boundaries. Content moderation is High because the agent is client-facing. The disabled capabilities exclude entire classes of behavior we don't want.

### Microsoft Entra authentication

Configured immediately as the agent's first setting (per Decision 3). Entra authentication means users sign in with their Microsoft 365 identity, and the agent receives their authenticated email address via `System.User.PrincipalName`.

The authentication is tenant-restricted - only users in the configured tenant can access the agent.

### Identity resolution flow

The agent needs to resolve the authenticated user's email to a Dataverse Contact (their personal record) and that Contact's parent Account (their organization). This resolution happens in **Conversation Start**, a special topic that fires once per conversation before any user-initiated topic.

The identity resolution invokes a helper flow `[Arcova Engage] Copilot - Resolve Contact by Email`. The flow takes an email address as input, looks up the matching Contact, retrieves the parent Account, and returns:

| Output | Type |
|---|---|
| UserEmail | String |
| UserContactId | String (GUID) |
| UserAccountId | String (GUID) |
| UserName | String |
| UserAccountName | String |

These return to the topic and are stored as **Global variables**, making them available across all topics for the conversation's duration.

### Dev mode toggle

A boolean variable `varDevMode` gates the identity resolution between two modes:

- **Production mode (varDevMode = false):** Email comes from `System.User.PrincipalName` automatically. The user never sees an identity prompt.
- **Dev mode (varDevMode = true):** The agent asks the user to enter an email address. This allows persona-switching during testing without needing actual M365 sign-ins for each test contact.

The dev mode toggle was essential for testing Phase 4. Seeded test contacts (John Rolzhausen, Simon Hamilton, Alice Katz, etc.) don't have real M365 accounts; dev mode lets the agent simulate them.

### Welcome greeting

After identity resolves, a personalized greeting fires:

> "Welcome, [UserName] from [UserAccountName]. I can help you check engagement status, submit new requests, view deliverables, or escalate urgent issues. What can I do for you?"

This greeting accomplishes three things: confirms the identity resolved correctly, sets expectations about the agent's capabilities, and invites the user to express their intent.

### Lessons from Section 2

**The helper flow uses the legacy "When Power Apps calls a flow" trigger, not the modern Copilot Studio trigger.** This is deliberate: the legacy trigger allows manual testing in Power Automate's test pane, which was useful for verifying the contact-resolution logic during build. The modern trigger (which Section 4-7's flows use) removes manual testing capability. For a flow as foundational as identity resolution, having manual test capability during development was worth the tradeoff.

---

## Section 3 - Knowledge Source and FAQ Topic

**Purpose:** Build a curated knowledge source (the `arc_faq` Dataverse table) and a topic that retrieves answers from it via semantic search, demonstrating Copilot Studio's knowledge integration capabilities.

### The arc_faq table

A new Dataverse table to hold FAQ content. Schema:

| Column | Type | Purpose |
|---|---|---|
| `arc_name` | Text | Question (primary column, displays in lookups) |
| `arc_answer` | Multiline Text | Answer body |
| `arc_category` | Choice (`arc_faqcategory`) | Category for content organization |

The `arc_faqcategory` global choice has 5 values (750740000-750740004): General, Process, SLA, Escalation, Submission. Categories are for organizational clarity; they're not used for retrieval logic (semantic search handles relevance).

**15 FAQ rows** were seeded across the five categories, with content drawn from the Project Brief's process descriptions, SLA configuration, and typical client questions.

### Rejected approaches worth documenting

Two approaches were considered and rejected, with rationale worth capturing:

**Rejected: keywords column.** An early proposal added a `arc_keywords` column for explicit tagging. This was rejected because Copilot Studio's knowledge integration uses **semantic retrieval** - the agent matches user questions to FAQ content based on meaning, not keyword overlap. A keywords column would duplicate effort and not improve match quality. The decision to omit it was deliberate.

**Rejected: escalation contact column on the FAQ table.** A proposal to include "if this question came up, who should the user contact?" data on each FAQ row. This was rejected because escalation logic belongs in the Escalate topic, not as static data per FAQ. Mixing escalation routing into knowledge data would conflate two concerns.

Both rejections are documented in PROJECT_BRIEF.md's Knowledge Source section.

### Knowledge source registration

The `arc_faq` table was added as a knowledge source in the Arcova Assist agent's settings. Configuration:

- **Source type:** Dataverse
- **Table:** arc_faq
- **Searchable columns:** Question and Answer (both)
- **Display name:** "Arcova FAQs"

Once added, semantic retrieval is configured automatically by Copilot Studio. No code or query logic to write.

### FAQ topic with description-driven discovery

The topic that consumes the knowledge source has a notable architecture decision: **no trigger phrases**. In current Copilot Studio versions, the FAQ topic discovery is entirely driven by the topic description. The orchestrator routes user queries to topics based on description-vs-intent matching.

The trigger description for the FAQ topic uses a seven-layer grounding defense:

1. **Positive scope statement** - what the topic IS for
2. **Example queries** - 5+ realistic user phrasings
3. **Negative carve-outs** - explicit "NOT for X (route to Y instead)" statements for each of the other 5 topics
4. **Pronoun resolution rule** - "'your'/'you' refers to Arcova Advisory's policies and processes, not the user's personal engagements"
5. **Knowledge source binding** - explicit reference to the arc_faq table as the answer source
6. **Citation guidance** - the agent should cite which FAQ row sourced its answer
7. **Out-of-scope handling** - what to do if the question isn't covered

This pattern - the multi-layered description with explicit negative carve-outs - became the template for all subsequent topic descriptions in Phase 4. Without the negative carve-outs, the orchestrator routed too many queries to FAQ that should have gone to other topics; with them, routing was clean.

### Trigger paradigm shift documentation

The original scoping (Section 1, Decision 1) called for "generative orchestration." During Section 3 build, an important platform reality surfaced: **current Copilot Studio versions removed trigger phrases entirely** for topics using generative orchestration. The only configurable trigger property is "The agent chooses" with its description-driven matching.

This was a departure from older Copilot Studio guidance that emphasized trigger phrase curation. The shift to pure description-driven discovery means topic descriptions carry the entire weight of intent routing - which is why the seven-layer grounding approach matters.

### Lessons from Section 3

**Description-driven discovery requires careful negative carve-outs across topics.** The orchestrator routes based on semantic similarity to descriptions. Without explicit "NOT for X, route to Y" statements naming the other topics, edge cases route ambiguously. Every topic in Phase 4 (Sections 4-7) follows this same multi-layered description pattern.

**Semantic retrieval handles synonyms and rephrasing well; explicit keywords are unnecessary.** A user asking "How long does it take to process my request?" finds an FAQ titled "What's your SLA?" through semantic similarity. Adding keyword tags would have been redundant effort.

---

## Section 4 - Check Engagement Status

**Purpose:** Build the first authored topic - a topic that lets a user ask "what's the status of my engagement?" and receive an adaptive card showing the engagement's details. This section is treated comprehensively because it was the hardest section of Phase 4 and established the architectural patterns that Sections 5-7 reuse.

### What was built

Three deliverables:

1. **Helper flow** `[Arcova Engage] Copilot - Get User Engagements` - queries Dataverse for the user's active engagements, resolves assigned agent names, and returns a structured response
2. **Check Engagement Status topic** - description-driven discovery, three-way conditional logic for the engagement count, and disambiguation when the user has multiple engagements
3. **Engagement Status adaptive card** - a Formula-format card displaying engagement details with formatted dates and graceful fallbacks for unassigned engagements

### Helper flow architecture

The flow uses the **modern "When Copilot Studio calls a flow" trigger** (vs. the legacy Power Apps trigger used in Section 2). The modern trigger is required for clean integration with topics but removes the ability to test the flow manually in Power Automate's test pane - flows must be invoked through the topic to be exercised.

Flow input: `AccountId` (text).

Flow structure:

1. **Get Active Engagements** - List rows on Engagements, filter `_arc_account_value eq [AccountId] and arc_status ne 750740005` (excludes Closed). Sort by SLA Due Date ascending, row count 10.
2. **Initialize varEngagements** - String array variable to accumulate the engagement records
3. **Initialize varEngagementsList** - String variable for the formatted display list
4. **Initialize varCounter** - Integer for numbering the display list
5. **Apply to each (engagement)** - inside:
   - **Condition: Check if Agent Assigned** - tests `_arc_assignedagent_value` is not null
   - **If yes: Get Agent Name** - List rows on Users filtered by the GUID to retrieve `fullname`
   - **If no:** empty branch
   - **Append Engagement Summary** - appends a JSON record to varEngagements with all fields including `AssignedAgentName` resolved via inline `if(empty(...))` expression
   - **Increment Counter** - varCounter += 1
   - **Append List Line** - appends `varCounter. EngagementTitle (Status)\n` to varEngagementsList
6. **Respond to Copilot** - returns three outputs: EngagementCount (number), EngagementsJson (text), EngagementsList (text)

The flow does meaningful data shaping. It doesn't just pass query results back - it resolves lookup GUIDs to display names, builds a pre-formatted display list, and serializes everything for easy topic-side consumption.

### Lookup column resolution pattern

Dataverse lookup columns return only the GUID, not the related record's primary column. To display "John Rolzhausen" as the assigned agent, the flow must look up the agent's full name via a follow-up query. The Condition-guarded lookup pattern handles both the populated and unpopulated cases:

- Populated: get the agent's `fullname`, use it in the JSON
- Unpopulated: fall through with empty string; the topic will display "Awaiting assignment"

This pattern became reusable across Phase 4. Section 5's deliverables flow uses the same approach for parent engagement names. Section 6's submit flow uses it for the post-routing assigned agent name.

### Topic architecture

The topic uses description-driven discovery (same pattern as Section 3) with negative carve-outs explicitly routing engagement-status questions away from other topics. The topic body:

1. **Call helper flow** with `Global.UserAccountId`
2. **Parse value** node converting `EngagementsJson` (text) to a typed table `Topic.Engagements`
3. **Three-way conditional** on `Topic.EngagementCount`:
   - = 0: empty-state message ("You don't have any active engagements right now...") and end
   - = 1: set `Global.SelectedEngagement = First(Topic.Engagements)` directly
   - else (multi): disambiguation logic
4. **Disambiguation** (multi-engagement branch):
   - Display the formatted list (Topic.EngagementsList from the flow)
   - Ask user to enter the number
   - Set `Global.SelectedEngagement = Index(Topic.Engagements, Topic.UserSelection)`
5. **Convergence:** both single and multi branches converge to a shared display node
6. **Display the Formula adaptive card** using `Global.SelectedEngagement.*` references
7. **End current topic**

### The Formula adaptive card

The card displays engagement details. After significant exploration of JSON-format card binding (which encountered the platform constraints documented in Lessons Learned), the working approach was the **Formula card format**:

```powerfx
{
    '$schema': "https://adaptivecards.io/schemas/adaptive-card.json",
    type: "AdaptiveCard",
    version: "1.5",
    body: [
        {
            type: "Container",
            style: "emphasis",
            items: [
                { type: "TextBlock", text: Global.SelectedEngagement.EngagementTitle, size: "Large", weight: "Bolder" }
            ]
        },
        {
            type: "FactSet",
            facts: [
                { title: "Status", value: Global.SelectedEngagement.Status },
                { title: "Priority", value: Global.SelectedEngagement.Priority },
                { title: "Category", value: Global.SelectedEngagement.Category }
            ]
        },
        {
            type: "FactSet",
            facts: [
                { title: "Submitted", value: Text(DateTimeValue(Global.SelectedEngagement.SubmittedDate), "mmmm d, yyyy") },
                { title: "SLA Due", value: Text(DateTimeValue(Global.SelectedEngagement.SLADueDate), "mmmm d, yyyy h:mm AM/PM") },
                { title: "Assigned Agent", value: If(IsBlank(Global.SelectedEngagement.AssignedAgentName), "Awaiting assignment", Global.SelectedEngagement.AssignedAgentName) }
            ]
        },
        {
            type: "Container",
            items: [
                { type: "TextBlock", text: "Description", weight: "Bolder", spacing: "Medium" },
                { type: "TextBlock", text: Global.SelectedEngagement.Description, wrap: true }
            ]
        }
    ]
}
```

Three notable patterns in this code:

1. **Direct Power Fx variable references** - `Global.SelectedEngagement.EngagementTitle` resolves at runtime through Power Fx's evaluation engine, not through adaptive card templating
2. **Date conversion via `Text(DateTimeValue(stringDate), "format")`** - dates cross the flow-to-topic boundary as strings (because the JSON serialization in the helper flow stringifies them); they must be parsed back to datetime values before formatting
3. **Empty-case fallback inline** - `If(IsBlank(...), "Awaiting assignment", value)` handles unassigned engagements without requiring a separate empty branch in the topic

### Why this took a long time

Section 4 was the most time-consuming section of Phase 4 because multiple platform constraints surfaced sequentially. The build path encountered:

- The modern flow trigger removes manual testing (workaround: invoke via topic with hardcoded inputs)
- Dataverse lookup columns return GUIDs only (workaround: Condition-guarded follow-up lookups)
- The Power Automate run history OUTPUTS panel shows truncated state and can mislead iteration debugging (workaround: verify via downstream action INPUTS)
- The "Ask with adaptive card" node requires Action.Submit and a YAML-format output schema (workaround: use "Send a message - Adaptive card" for display-only cards)
- JSON adaptive card variable binding via `${variable}` placeholders has no clean UI surface (workaround: Formula card format with Power Fx references)
- Power Fx's `Concat(AddColumns(Sequence(N), ...), ...)` idiom is unsupported in Copilot Studio (workaround: build the numbered list in the flow as a string)
- Topic-scoped variables don't resolve reliably at branch convergence points (workaround: promote to Global scope)
- Date fields cross flow boundaries as strings, not datetimes (workaround: wrap with DateTimeValue in the card)

Each of these required diagnosis and adaptation. Each became a reusable pattern for subsequent sections. By the time Sections 5-7 were built, none of these consumed significant additional time.

### Test scenarios verified

Six scenarios from the test matrix:

1. **Single engagement** - Simon Hamilton at Test Account 2, one active engagement assigned to Cindy Rolzhausen - no disambiguation, direct card render
2. **Multi-engagement disambiguation** - John Rolzhausen at Northpoint, 5 engagements - numbered list display, user picks number
3. **Multi-engagement card after selection** - same persona, after picking - card renders with the selected engagement's data
4. **Zero engagements** - Alice Katz at Test Account - friendly empty-state message, no card
5. **Escalated engagement** - picking an engagement with Status = Escalated - card renders with "Escalated" status (no separate badge; the status communicates escalation)
6. **Unassigned engagement** - picking a New engagement with no agent - card displays "Awaiting assignment" via the IsBlank fallback

All six scenarios passed. The escalation badge was scoped but deferred to Phase 5 with rationale documented (the Status field already communicates escalation; conditional `isVisible` rendering in Formula cards is unproven and high-risk).

### Lessons from Section 4

Section 4 generated the highest-value Lessons Learned content of Phase 4. The headline lessons are consolidated in the cross-cutting Lessons Learned section near the end of this document. The section-specific points:

**The "explore JSON cards, settle on Formula cards" arc is a real engineering narrative, not a retreat.** Multiple binding approaches were tried because adaptive cards are explicitly tested by AB-410 and worth investing in. The Formula card discovery wasn't a workaround - it's the natively supported binding mechanism that JSON cards lack in this Copilot Studio version.

**The shared-convergence display pattern is cleaner than per-branch duplication.** Originally, each branch (single, multi) had its own card display node. The refactor to a shared convergence node, fed by a Global variable set in each branch, reduced duplication and made future card refinements single-source.

---

## Section 5 - List My Deliverables

**Purpose:** Build a topic that returns the user's active deliverables across all their account's engagements, displayed as a formatted numbered list.

### What was built

Two deliverables:

1. **Helper flow** `[Arcova Engage] Copilot - Get User Deliverables` - queries deliverables filtered through the engagement-to-account relationship
2. **List My Deliverables topic** - simpler than Section 4 (no disambiguation, no card), just count-based conditional and list display

### Architectural choices

Several decisions taken at scoping (Section 1 of Section 5's build):

- **Account scope** (consistent with Section 4) - the user sees deliverables for any engagement on their account
- **Active deliverables only** - excludes Delivered (750740003) and Cancelled (750740004) statuses; users asking about deliverables care about what's pending
- **Formatted text list** rather than adaptive card - lists of items don't benefit from card styling the way a single record's details do
- **Flat list with engagement names inline** - each line shows the deliverable name plus its parent engagement name, eliminating the need for per-engagement disambiguation
- **Friendly empty-state** matching Section 4's pattern

### The two-step filter pattern

Deliverables don't have a direct account link - they link to engagements, which link to accounts. To filter deliverables by account, the flow uses a two-step approach:

1. **Get Account Engagements** - List rows on Engagements filtered by `_arc_account_value`, returns the engagement IDs
2. **Build a dynamic OR-filter** - constructs `_arc_engagement_value eq id1 or _arc_engagement_value eq id2 or ...` by iterating through the engagement IDs and appending fragments to a string variable
3. **Trim the trailing ` or `** - the build loop appends `" or "` after each fragment, so the final string needs cleanup
4. **Get Active Deliverables** - List rows on Deliverables filtered by the composed OR-string AND status exclusions

A relationship-traversal alternative (`arc_Engagement/_arc_account_value eq [AccountId]`) was attempted first but failed because the navigation property name isn't guessable - it depends on how the relationship was set up in Phase 0 and isn't documented in any UI surface. The two-step pattern is more verbose but uses only standard List rows queries with no schema-specific navigation knowledge required.

### The empty-engagement guard

A defensive Condition guards the deliverables query: if the account has zero engagements (varEngagementFilter is empty), skip the trim/query/list-build entirely. Without this guard, `substring("", 0, -4)` throws a runtime error.

This guard is essential because zero-engagement accounts are valid (a brand-new client's account with no engagements yet) and the topic must handle them gracefully. The guard also enables the topic's zero-deliverables empty-state message to fire from the same flow output rather than requiring two different invocation paths.

### Power Automate variable scoping constraint

Power Automate requires `Initialize variable` actions to live at the top level of the flow - not inside Conditions, scopes, or loops. This constraint forced an architectural pattern: all three variables (varEngagementFilter, varDeliverablesList, varCounter) initialize at the top of the flow with safe default values, then conditional logic later either populates them or leaves them at defaults. The deliverables flow ends with the variables having either populated data (deliverables exist) or default values (no engagements, no deliverables to enumerate). Both cases return cleanly through Respond to Copilot.

### Topic structure

Significantly simpler than Section 4:

1. Call the helper flow
2. Condition: `DeliverableCount = 0` - friendly empty-state message - end
3. Else: display `{Topic.DeliverablesList}` - end

No parse-value, no disambiguation, no card. Just two Send a message nodes and a Condition.

### Test scenarios verified

Three scenarios from the test matrix:

1. **Multi-deliverable (Northpoint)** - 4 active deliverables across 4 different engagements - formatted numbered list with engagement names
2. **Zero-deliverable (Simon Hamilton at Test Account 2)** - one engagement, no deliverables on it yet - friendly empty-state
3. **Zero-deliverable (Alice Katz at Test Account)** - zero engagements, therefore zero deliverables - friendly empty-state via the empty-engagement guard path

### Lessons from Section 5

**The two-step filter pattern (engagements first, then deliverables) is more robust than relationship-traversal.** Relationship navigation in OData filters requires knowing the relationship's navigation property name, which isn't readily discoverable. Splitting into two explicit queries avoids the guesswork and is easier to debug.

**Variable initialization with `""` in the Value field inserts literal double-quote characters, not an empty string.** A long debugging session traced filter syntax errors to a single quote-pair character in the variable's initial value. The fix was deleting them and leaving the field truly empty.

---

## Section 6 - Submit New Engagement

**Purpose:** Build the most complex authored topic - a multi-turn slot-filling conversation that creates a new engagement in Dataverse, with a confirmation card showing the submitted engagement plus Phase 3-populated SLA Due Date and Assigned Agent.

### What was built

Two deliverables:

1. **Helper flow** `[Arcova Engage] Copilot - Submit New Engagement` - creates the Dataverse engagement row, waits for downstream Phase 3 flows to populate routing fields, re-fetches, and returns the enriched data
2. **Submit New Engagement topic** - 5-question slot-filling conversation with confirmation, calling the flow and displaying a result card

### The slot-filling pattern

The topic asks 5 questions in sequence:

1. Engagement title (free text)
2. Description (free text)
3. Category (multiple choice: Advisory, Technology, Transformation, Compliance, Other)
4. Priority (multiple choice: Low, Medium, High, Critical)
5. Additional notes (free text, with `(none)` handling for skip-style responses)

After all questions, a confirmation summary displays the collected data and asks a final "Yes, submit it / No, cancel" question. Only on Yes is the flow invoked.

### The OptionSetValue conversion problem

Copilot Studio's multiple-choice question stores responses as **OptionSetValue** (not Text). When the flow expects a Number for the Dataverse choice field, the OptionSetValue can't be passed directly. Two conversion approaches were tested:

- **`Topic.Category.Value`** - the .Value property should return the numeric option set value; rejected by this Copilot Studio version
- **`Switch(Text(Topic.Category), "Advisory", 750740000, ...)`** - works reliably; converts the label to text first, then maps to numeric

The Switch pattern is more verbose but explicit and self-documenting. Two Set variable nodes (one for Category, one for Priority) convert the user's choice labels to the numeric values expected by Dataverse.

### The 5-way auto-generated branches

Each multiple-choice question auto-generates downstream Condition branches - one per option, plus an "All other conditions" catch-all. For the Submit topic's purpose (slot-filling toward a single submission), these branches are unnecessary. The build deleted them all and connected each Question directly to the next, producing a clean linear flow.

### The helper flow architecture

The flow creates the engagement and then waits for Phase 3's flows to do their downstream work:

1. **Create Engagement** - Add a row to Engagements with the user's inputs plus hardcoded Status (750740000 = New) and Escalated (false). Lookups for Client Organisation and Requester are set via `@odata.bind` syntax using `Global.UserAccountId` and `Global.UserContactId`.
2. **Delay** - 8 seconds. Gives Phase 3 Section 2 (Auto-populate Submitted Date) and Phase 3 Section 5 (Engagement Intake Router) time to fire and populate downstream fields.
3. **Get Engagement After Routing** - Get a row by ID, re-fetches the engagement with all fields including the now-populated SLA Due Date and Assigned Agent.
4. **Check if Agent Assigned + Get Agent Name** - same Condition-guarded lookup pattern from Section 4, resolves the assigned agent's full name when available.
5. **Respond to Copilot** - returns the engagement details with dates formatted via `convertFromUtc(..., 'Eastern Standard Time', ...)` to avoid timezone confusion in the display.

### Cross-flow composition - the v1.0 limitation

The flow's Respond outputs include Assigned Agent name. When a user submits an Advisory engagement (mapped to John Rolzhausen in `arc_categoryassignment`), the Phase 3 Intake Router does assign John - but the assignment is often not visible during the 8-second delay window. Even extending to 30 seconds didn't reliably catch the assignment in the re-fetch.

Three options were considered:

- Extend the delay further (rejected - unbounded wait risk, degraded UX)
- Implement polling (rejected - same unbounded wait risk)
- Accept transient "Awaiting assignment" display in the card (selected)

The Assigned Agent IS populated correctly in Dataverse (verified in the Agent Hub) and IS included in the Phase 3 intake confirmation email. The card just shows the transient state at the moment of render. This is an explicit v1.0 tradeoff prioritizing fast confirmation render over agent name accuracy at that surface.

### The confirmation card

A Formula adaptive card with `style: "good"` (green/positive treatment) to signal submission success. The card shows the engagement title, status/priority/category, submitted date, SLA due date, assigned agent (with the "Awaiting assignment" fallback), and a closing line about email confirmation.

### Lessons from Section 6

**Multiple-choice questions store OptionSetValue, not Text.** Direct comparisons to text labels fail with type errors. The reliable pattern is `Switch(Text(optionSetVar), "Label1", value1, ...)` for label-to-value conversion.

**Power Automate `if()` does not short-circuit evaluation.** Both branches of an inline `if()` are evaluated even when the condition is false. Defensive null guards must check the underlying nullity of every referenced output, not just the condition. Section 6 hit this when the Respond to Copilot expression for AssignedAgentName tried to dereference `Get_Agent_Name`'s output even when the Condition guarding it was false. The fix: extract conditional values into variables inside the branches, then reference the variables from Respond.

**Dataverse stores datetime values in UTC; flows return UTC strings unless explicitly converted.** A 4-hour offset between the Agent Hub display (local time) and the Copilot card display (UTC) is the visible symptom. The fix is `convertFromUtc(value, 'Eastern Standard Time', format)` in the flow.

**Single-word answers like "skip" can misroute through the generative orchestrator.** During testing, "skip" as an answer to the optional Notes question was intercepted by the orchestrator and routed to the system Escalate topic. The fix was rewording the question to use "none" or "type a dash" instead. Command-like words are particularly prone to false-positive intent matches.

---

## Section 7 - Escalate an Issue

**Purpose:** Build the simplest authored topic of Phase 4 - one that demonstrates the highest-weighted AB-410 exam skill (triggering external flows from Copilot Studio) by invoking Phase 3 Section 8's existing HTTP-triggered escalation flow.

### What was built

One deliverable:

1. **Escalate an Issue topic** - reuses Section 4's helper flow for engagement selection, asks for an escalation reason, then makes an HTTP POST to Phase 3's escalation flow

No new helper flow. This is intentional - the topic demonstrates cross-phase composition by invoking existing infrastructure without modification.

### Topic structure

1. Welcome message
2. Call `[Arcova Engage] Copilot - Get User Engagements` (the Section 4 flow, reused)
3. Parse value, three-way conditional, disambiguation - same pattern as Section 4
4. Question: escalation reason
5. Confirmation summary + Yes/No question
6. On Yes: **HTTP POST** to the Phase 3 escalation flow with `engagementId` and `escalationReason` in the JSON body
7. Success message - end
8. On No: cancel message - end

### The HTTP body interpolation issue

A meaningful debugging moment: when the JSON body was authored with literal-quoted variable references:

```json
{
  "engagementId": "{Global.SelectedEngagement.EngagementId}",
  "escalationReason": "{Topic.EscalationReason}"
}
```

...the variables didn't resolve. The body was sent to Phase 3 with the literal text `{Global.SelectedEngagement.EngagementId}` as the engagementId value, causing Phase 3's Get Engagement to fail with a 404 (no record matches that string as a GUID).

The fix was authoring the body as a Power Fx expression in formula mode, with variable references outside the quote wrapping. Different Copilot Studio versions surface this differently - some have a formula/expression mode toggle on the body editor, others use `@{}` syntax for runtime resolution.

### The HTTP timeout issue

After the body interpolation was fixed, a different problem surfaced: the HTTP call timed out after 30 seconds. The Phase 3 escalation flow includes a "Post adaptive card and wait for response" action that suspends the flow until the senior agent acknowledges in Teams. That suspension could be minutes or hours - far longer than Copilot Studio's 30-second HTTP timeout.

The fix was an architectural change to the Phase 3 flow: **move the Respond to a request action earlier in the flow**, immediately after the Update Engagement action, before the Teams card action. Copilot now receives a fast response (within 1-2 seconds), and the Teams acknowledgment workflow continues asynchronously in the background.

This is the **fire-and-forget pattern** for HTTP-triggered flows with downstream long-running actions: acknowledge quickly, complete critical data updates synchronously, perform notifications and approvals asynchronously.

### Test scenarios verified

The happy path was tested end-to-end:

1. User: "I need to escalate"
2. Disambiguation: pick "Supply Chain Visibility Platform Design"
3. Reason: "CEO needs this immediately"
4. Confirm: Yes
5. Engagement marked Escalated = Yes in Dataverse (immediately)
6. Escalation reason populated (immediately)
7. Success message in chat (within 2 seconds)
8. Teams adaptive card sent to senior agent (asynchronously)
9. Email sent to senior agent (asynchronously)
10. Audit Communication created (asynchronously)

Cancel path and single/zero-engagement edge cases were verified as well, all behaving correctly.

### Lessons from Section 7

**HTTP body JSON: literal-quoted variable references don't resolve.** Writing `{"field": "{Topic.Variable}"}` produces a literal string. The fix is formula mode or `@{}` syntax for runtime expression evaluation.

**The fire-and-forget pattern is essential for HTTP-triggered flows with long-running downstream actions.** Move the Respond action to fire immediately after critical updates; perform notifications and approvals asynchronously to avoid blocking the HTTP requester.

---

## Architectural Patterns Established

Phase 4 produced several reusable patterns that apply beyond the four authored topics. These are worth documenting separately because they're the architectural backbone of the agent and would carry forward to any future Copilot Studio work on this project.

### Pattern 1 - The helper flow with modern Copilot Studio trigger

All Phase 4 helper flows use the "When Copilot Studio calls a flow" trigger (except Section 2's identity resolution, which uses the legacy trigger for testability reasons). The trigger removes manual testing capability but provides clean topic integration with strongly-typed inputs and outputs. The mitigation for the testing limitation: wire the flow into a topic with hardcoded test inputs early, then exercise it through the topic.

### Pattern 2 - Condition-guarded lookup for related-record names

Dataverse lookup columns return GUIDs, not display names. To show "John Rolzhausen" instead of `12345abc-...`, the flow performs a follow-up List rows query to retrieve the related record's name. The lookup is wrapped in a Condition that checks whether the lookup value is populated:

- If yes: query for the name, use it
- If no: fall through with empty string, let the topic display a fallback

This pattern appears in Sections 4 (assigned agent), 5 (parent engagement), and 6 (post-routing agent). It's the canonical pattern for lookup display in Copilot Studio flows.

### Pattern 3 - Flow-builds-the-list for iteration

Copilot Studio's Power Fx implementation doesn't support all Canvas app iteration idioms. Patterns like `Concat(AddColumns(Sequence(N), ...), ...)` fail with "Identifier not recognized" errors. The reliable workaround is to build iterative strings (numbered lists, formatted summaries) in the helper flow using `Initialize variable`, `Apply to each`, `Increment variable`, and `Append to string variable` actions, then return the pre-built string as a flow output.

The topic side becomes trivial: reference the flow's pre-built string variable. All iteration complexity lives in the flow, where Power Automate's expression language handles it reliably.

### Pattern 4 - Formula card format for adaptive cards

The architecturally correct binding mechanism for adaptive cards in current Copilot Studio is the Formula card format - a Power Fx record literal where variable references resolve at runtime through the expression engine. JSON cards with `${variable}` placeholders have no clean binding surface in this version of the platform. The Formula card discovery was a meaningful pattern win for Phase 4 because it unblocked Section 4's status card and Section 6's confirmation card.

### Pattern 5 - Global variable scope for cross-branch state

Variables set in multiple branches and read at a shared downstream node should be Global-scoped, not Topic-scoped. Topic-scoped variables don't reliably resolve at branch convergence points. The shared-display convergence pattern (both branches set the same variable, one downstream node consumes it) is cleaner than duplicating the consuming node in each branch - and Global scope is what makes it work.

### Pattern 6 - The seven-layer trigger description

All Phase 4 topics use the same trigger description pattern:

1. Positive scope statement
2. Example user phrasings (5+)
3. Negative carve-outs (NOT for X, route to Y) naming each other topic explicitly
4. Pronoun resolution rule (when applicable)
5. Data source binding (when applicable)
6. Output expectations
7. Out-of-scope handling

This pattern makes the orchestrator's routing reliable. Without negative carve-outs in particular, ambiguous queries route inconsistently.

### Pattern 7 - The fire-and-forget HTTP response

HTTP-triggered flows invoked from Copilot Studio must respond within the 30-second timeout. For flows with downstream long-running actions (Teams card waits, approval requests), the Respond action must fire after critical data updates but before the long-running work. The long-running work continues asynchronously, decoupled from the HTTP requester's wait.

---

## Lessons Learned (Curated)

The most instructive lessons from Phase 4, distilled to the 12 most reusable across future work.

### 1. The modern Copilot Studio flow trigger removes manual flow testing

The "When Copilot Studio calls a flow" trigger is required for clean topic integration but eliminates the ability to test flows through Power Automate's test pane. The mitigation: wire flows into topics with hardcoded test inputs early, then exercise through the topic. The legacy trigger preserves manual testing but doesn't integrate as cleanly with current Copilot Studio - the right choice depends on whether testability or topic integration is more important for the flow in question.

### 2. Dataverse lookup columns return GUIDs only; resolution requires follow-up queries

The lookup value is just the GUID of the related record. To display anything human-readable about the related record (name, email, etc.), the flow must execute a follow-up List rows or Get a row by ID query. The Condition-guarded lookup pattern handles both populated and unpopulated cases gracefully.

### 3. The Power Automate run history OUTPUTS panel shows truncated state

When inspecting an Apply to each loop's variable-appending action, the INPUTS panel correctly shows per-iteration data, but the OUTPUTS panel shows the variable's accumulated state truncated. This can mislead debugging by suggesting iterations are producing duplicate or stale data. The reliable verification: inspect the next downstream action's INPUTS, which shows the variable's final state after all iterations complete.

### 4. "Run After = failed" hides errors rather than handling them

Configuring an action to run after a previous action's failure isn't error handling - it's error masking. The proper pattern is explicit conditional logic that anticipates failure modes (empty results, missing related records, type mismatches) and provides defensive defaults or fallback display values.

### 5. Adaptive card variable binding requires the Formula card format

Current Copilot Studio versions do not expose a clean binding mechanism for JSON-format adaptive cards with `${variable}` placeholders. The Formula card format - a Power Fx record literal with direct variable references - is the natively-supported pattern. Cards built in Formula format render identically to JSON-format cards but resolve variables through the Power Fx expression engine without requiring schema editors or sample data panels.

### 6. Date fields cross flow-to-topic boundaries as strings

When a helper flow appends date fields to a JSON array variable, the dates are serialized as quoted strings. The Parse value node in the topic types them as strings, not datetimes. Formatting dates in the topic's adaptive card therefore requires `Text(DateTimeValue(stringDate), "format")` - parse the string back to a datetime, then format. Alternatively, format dates in the flow using `formatDateTime()` and return ready-to-display strings; this is the cleaner approach when the topic doesn't need to manipulate the dates further.

### 7. Copilot Studio's Power Fx does not support all Canvas app idioms

Patterns like `Concat(AddColumns(Sequence(N), ...), ThisRecord.Value & ...)` work in Canvas apps but produce "Identifier not recognized" errors in Copilot Studio. The reliable workaround for iterative string building is to do the iteration in the helper flow using standard Power Automate actions (Initialize variable, Apply to each, Append to string variable), then return the pre-built string to the topic. "Do iteration in the flow, do display in the topic" is the safest default.

### 8. Initialize variable Value field literals are interpreted as text, not as empty

A String variable initialized with `""` in the Value field contains two literal double-quote characters, not an empty string. To create a truly empty String variable, the Value field must be left completely blank. This caused several downstream issues across Phase 4 builds - filter strings starting with `""` produced cryptic OData syntax errors.

### 9. Variables read at branch convergence points should be Global, not Topic

Topic-scoped variables set inside one branch of a Condition may not reliably resolve at downstream nodes outside the branch. Promoting to Global scope makes the variable readable anywhere in the topic regardless of which branch wrote it. This is the architectural prerequisite for the shared-display convergence pattern (both branches set the same variable, one downstream node consumes it).

### 10. Power Automate `if()` does not short-circuit evaluation

Both branches of an inline `if()` expression are evaluated even when the condition is false. Defensive null guards must check the underlying nullity of every referenced output, not just the condition. When an action runs conditionally (inside an If Yes branch), referencing its output from a downstream Respond expression will fail when the condition was false. The reliable pattern is extracting conditional values into variables inside the branches, then referencing the variables from downstream actions.

### 11. Dataverse stores datetime values in UTC

Power Automate flows that use `formatDateTime()` directly on Dataverse date fields return UTC-formatted strings. Model-driven apps and Canvas apps display in local time, creating visible discrepancies. The fix in flows is `convertFromUtc(value, 'Eastern Standard Time', format)`. For multi-tenant scenarios, the timezone should be sourced from user locale rather than hardcoded.

### 12. HTTP-triggered flows must respond within Copilot Studio's 30-second timeout

Long-running flow actions (Teams card waits, approval requests) suspend the flow and prevent the HTTP response from firing. Copilot Studio surfaces this as `HttpRequestTimeout`. The fix is the fire-and-forget pattern: move the Respond action to fire immediately after critical data updates, then perform long-running actions asynchronously. The Copilot user gets fast confirmation; downstream workflows continue without blocking.

---

## Cross-Flow Composition

Phase 4's most distinctive narrative is that **a single Copilot conversation triggers a coordinated cascade across the entire project's infrastructure**. The Submit New Engagement topic demonstrates this most completely:

1. User submits via Copilot conversation
2. **Section 6 helper flow** creates the engagement row
3. **Phase 3 Section 2** (Auto-populate Submitted Date) fires automatically, sets Submitted Date
4. **Phase 3 Section 5** (Engagement Intake Router) fires automatically:
   - Looks up SLA hours from `arc_slaconfig` based on category and priority
   - Calculates SLA Due Date
   - Looks up the default agent from `arc_categoryassignment`
   - Updates the engagement with both
   - Sends an intake confirmation email
   - Creates an audit Communication record
5. **Phase 1 Business Process Flow** activates automatically, starts tracking the engagement through its lifecycle
6. **Section 6 helper flow** re-fetches the engagement and returns enriched data to the topic
7. **Section 6 topic** displays the confirmation card

Six pieces of infrastructure across four phases, all working together because each was designed with the others in mind. Phase 0's schema established the data shape. Phase 1's BPF auto-activates on row creation. Phase 3's flows handle downstream automation. Phase 4 sits on top as the conversational interface.

The same cross-phase composition appears in Section 7 (Escalate an Issue), where the topic invokes Phase 3 Section 8's HTTP-triggered escalation flow without any modification beyond the architectural reordering of the Respond action. The escalation flow was designed in Phase 3 to be externally invokable; Section 7 is the proof of that design.

This composition is the portfolio's strongest demonstration of architectural thinking. The phases didn't build isolated capabilities - they built a coordinated system where each layer extends and consumes the layers beneath it.

---

## Phase 4 Completion Criteria

### Agent and authentication
- [x] Arcova Assist agent created within the Arcova Engage solution
- [x] Microsoft Entra authentication configured, tenant-restricted
- [x] Content moderation High, connected agents disabled, file uploads disabled, web search disabled
- [x] Identity resolution flow (`[Arcova Engage] Copilot - Resolve Contact by Email`) returning UserEmail, UserContactId, UserAccountId, UserName, UserAccountName as Global variables
- [x] Dev mode toggle (`varDevMode`) enabling persona-switching during testing
- [x] Personalized welcome greeting referencing UserName and UserAccountName

### Knowledge source and FAQ topic
- [x] `arc_faq` table created with question, answer, and category columns
- [x] 15 FAQ rows seeded across 5 categories (General, Process, SLA, Escalation, Submission)
- [x] Knowledge source registered in agent settings
- [x] FAQ topic with seven-layer grounding description for reliable orchestration
- [x] Semantic retrieval validated with test queries across all categories

### Authored topics (Sections 4-7)
- [x] Check Engagement Status topic with Formula adaptive card, three-way conditional, multi-engagement disambiguation
- [x] List My Deliverables topic with formatted text list, account-scoped via two-step filter
- [x] Submit New Engagement topic with multi-turn slot-filling, OptionSetValue conversion, confirmation card
- [x] Escalate an Issue topic with HTTP-triggered flow invocation, fire-and-forget pattern

### Helper flows
- [x] `[Arcova Engage] Copilot - Get User Engagements` (Section 4, reused by Section 7)
- [x] `[Arcova Engage] Copilot - Get User Deliverables` (Section 5)
- [x] `[Arcova Engage] Copilot - Submit New Engagement` (Section 6)

### Phase 3 flow updates
- [x] `[Arcova Engage] Engagement - HTTP Escalation` restructured: Respond action moved above Teams card action (fire-and-forget pattern)

### Cross-phase composition verified
- [x] Section 6 submission triggers Phase 3 Section 2 (Submitted Date), Phase 3 Section 5 (Intake Router), Phase 1 BPF activation
- [x] Section 7 escalation triggers Phase 3 Section 8 (Engagement update, Teams card, email, audit Communication)
- [x] Identity resolution Global variables consumed by all four authored topics

### Architectural decisions documented
- [x] Section 1 - eight original scoping decisions
- [x] Section 1 - two decisions evolved during build (adaptive card format, helper-flow-everywhere)
- [x] Section 1 - two decisions added during build (account scoping, global variables)
- [x] Escalation badge deferred to Phase 5 with rationale
- [x] Assigned Agent in Submit card deferred to Phase 5 with rationale (race condition with Phase 3 Intake Router)
- [x] Channel = Copilot in audit Communications deferred to Phase 5

### Documentation
- [x] PHASE_4.md committed to GitHub
- [x] Screenshots captured across all sections committed to `/screenshots/phase4/`

---

## Verification Artifacts

Screenshots committed to `/screenshots/phase4/`:

| Range | Content |
|---|---|
| 01-15 | Section 2 (agent shell, Entra auth, identity resolution flow, dev mode) |
| 16-28 | Section 3 (arc_faq table, knowledge source registration, FAQ topic with description) |
| 29-50 | Section 4 (helper flow with lookup pattern, parse value, conditional logic, Formula adaptive card, six test scenarios) |
| 51-66 | Section 5 (helper flow with two-step filter, empty-engagement guard, topic structure, three test scenarios) |
| 67-85 | Section 6 (helper flow with create+delay+refetch, multi-turn slot-filling topic, OptionSetValue conversion, confirmation card with style:good) |
| 86-98 | Section 7 (topic structure, HTTP request node configuration, Phase 3 flow restructure, end-to-end test) |

---

**Phase 4 complete.** Arcova Assist is built, tested, and integrated with the rest of the Arcova Engage solution. The agent provides conversational access to engagement status, deliverables, submission, and escalation - all sitting on top of the Phase 0 schema, the Phase 1 BPF, the Phase 2 Canvas app patterns, and the Phase 3 flow infrastructure. Every cross-phase touchpoint visible in this phase was deliberately designed in earlier phases to be reused here.

Next: Phase 5 (Integration & Polish) embeds the agent in the Canvas App, polishes Phase 4 deferral items, and completes the README. Phase 6 (Power BI enhancements) is the post-certification stretch goal.

