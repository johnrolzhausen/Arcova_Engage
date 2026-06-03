# Phase 5: Integration & Polish

## Overview

Phase 5 is the final pre-certification phase. Where earlier phases each built a major surface (the Agent Hub, the Client Portal, the automation flows, the Copilot agent), Phase 5 is about making those surfaces work together as one coherent system and producing the documentation that presents the project as a portfolio piece. There is one substantial net-new build (embedding the Copilot agent in the Client Portal), one round of cross-surface integration testing, one timeboxed feature that was deliberately deferred, and the portfolio documentation set (personas, architecture diagram, README, and this retrospective).

The phase was scoped into seven sections, built in this order: architectural scoping, the Canvas App embed, end-to-end integration testing, the Closure Approval command bar button, the personas and architecture documents, the README, and this retrospective.

---

## Section 1: Architectural scoping

The headline deliverable of Phase 5 was embedding the Arcova Assist agent into the Client Portal's chat screen, replacing the placeholder that had been in place since Phase 2. Before building, the embed mechanism needed a decision, and research surfaced that the platform had shifted in a way that directly affected the original plan.

The built-in Copilot control for canvas apps, the obvious low-code path the Phase 2 placeholder had assumed, was deprecated. It could no longer be added to new canvas apps, and existing uses were on a path to becoming unsupported. Building the headline feature on a deprecated control was not a defensible portfolio choice. Microsoft's official successor, Microsoft 365 Copilot in canvas apps, was still in preview for canvas at the time and oriented toward structured data responses rather than hosting a custom Copilot Studio agent with its own authored topics. That left it unsuitable for hosting Arcova Assist as built.

The chosen path was the ChatControl PCF component from the Copilot Studio Samples repository, distributed as the AgentControls managed solution (v1.0.0.9). It is a Power Apps Component Framework control built on the Microsoft 365 Agents SDK that embeds a Copilot Studio agent using Bot Framework WebChat, and it supports single sign-on. This was the path that worked in production, hosted the actual authored agent, and delivered the seamless-identity requirement. Importantly, it is built on the Agents SDK, which is Microsoft's forward-looking integration layer, so the choice is not a dead-end workaround.

A cost check confirmed the approach required no monetary investment: the Azure app registration the control depends on lives in the permanent free Microsoft Entra ID tier, with no trial clock and no billable services involved. 

A note on guidance versus practice: Microsoft's reference architecture for the PCF-plus-Agents-SDK pattern recommends the Copilot sidecar for chat-style scenarios and reserves PCF controls for more structured, single-response integrations. The Phase 2 design had committed `scrChat` as a chat surface, so the WebChat-based ChatControl was used deliberately, with the trade-off documented rather than hidden. This is a reasoned exception to the guidance, not an oversight.

---

## Section 2: The Canvas App embed

This was the one substantial net-new build of the phase. The work fell into four parts: the Azure identity setup, the environment preparation, the import and placement, and a debugging cycle.

**Azure app registration.** A single app registration was created in Microsoft Entra ID, configured as single-tenant, with a single-page-application platform and the set of redirect URIs the control requires (the runtime and authoring URLs, plus the environment's own instance URL). The "Allow public client flows" setting was enabled, and one delegated API permission was added: Power Platform API, CopilotStudio.Copilots.Invoke. Admin consent was granted tenant-wide so users would not face a consent prompt. No client secret was needed; this is a native public client.

**The four identifiers.** The control needs four values, which were gathered and cross-verified across three independent sources (the Azure registration, the Power Apps session details, and the Copilot Studio agent metadata): the application (client) ID, the tenant ID, the environment ID, and the agent identifier (schema name `arc_ArcovaAssist`). The fact that three sources agreed on the shared values meant later debugging could rule out a wrong-ID cause with confidence.

**Environment preparation.** Two environment settings had to be in place before import: the attachment file-size limit was raised to accommodate the large solution file, and Power Apps component framework support for canvas apps was confirmed enabled. The AgentControls managed solution then imported cleanly as its own separate solution, kept distinct from the Arcova Engage solution so the boundary between authored work and imported tooling stays clear.

**Placement.** The Phase 2 placeholder card on `scrChat` was hidden (set to invisible rather than deleted, so it remains recoverable), and the ChatControl was placed inside the existing chat container. Its four identifier properties were set, the `username` property was left at its default of `User().Email` (this is the mechanism that carries the signed-in user's identity into the agent for single sign-on), and `agentTitle` was set to "Arcova Assist."

**The render-loop bug.** On first run, the control rendered but blinked continuously, stealing focus from the input box on every cycle so the chat could not be used. Initial investigation found that several string properties (`message`, `eventValue`, `styleOptions`) contained two literal quote characters rather than being genuinely empty, which can cause repeated sends; clearing them was necessary but did not stop the blink. The actual fix was setting `agentTitle` to a non-empty value. An empty `agentTitle` caused the control to re-render in a loop. The lesson is reproducible and worth recording: the ChatControl requires a non-empty `agentTitle`, and leaving it blank causes a render loop.

The result is a working embed: a requester opens the portal already signed in, navigates to the chat screen, and is greeted by name with the agent's full authored topics available, no second login.

---

## Section 3: End-to-end integration testing

With the embed working in isolation, integration testing verified that the four surfaces (Agent Hub, Client Portal, Copilot test pane, and the embedded agent) stay coherent with one another. The testing was deliberately focused rather than exhaustive: the cross-phase composition was already validated in Phases 3 and 4, so Phase 5 testing concentrated on the embed as a newly introduced production read-and-write surface. Five tests were run.

**A1, embed submit propagates everywhere.** An engagement submitted through the embedded agent appeared moments later in the Agent Hub with every routing field correctly populated: submitted date, calculated SLA due date, assigned agent, and an active business process flow. This is the deepest path in the system, a single conversational submission triggering a cascade across infrastructure from three different phases, and it passed end to end. It is the strongest single piece of evidence for the project's architectural thesis.

**A2, portal submit visible in embed.** An engagement submitted through the Client Portal's own Submit screen appeared in the embedded agent's results on a fresh query, confirming the embed reads live data regardless of where the record originated.

**A3, embed escalation reflected in Agent Hub.** Escalating an engagement through the embedded agent updated the record's status and reason in the Agent Hub and triggered the downstream status-notification flow, demonstrating the externally callable escalation path working through the embed.

**B1, identity consistency.** The same user resolved to the same identity (John Rolzhausen at Northpoint Logistics) across the embed and the portal, by different mechanisms: the embed via single sign-on, the portal via its dev persona switcher. They converge on the same identity.

**D1, security boundary.** A known engagement belonging to another account did not appear in the user's embedded results, confirming that account-scoping holds through the embed surface.

All five passed. A secondary finding emerged during testing: Adaptive Card container styles render with higher fidelity in the embedded WebChat surface than in the Copilot Studio authoring pane. A card styled to signal success showed a full colored band in the embed where the authoring preview had shown only a subtle tint.

---

## Section 4: Closure Approval command bar button (deferred)

This section was scoped as a modern command-bar button on the Agent Hub that would invoke the Phase 3 closure approval flow against the active engagement record. It was timeboxed, with a plan to defer to post-certification if it proved sticky.

Research at the start of the timebox surfaced a platform limitation rather than a configuration hurdle: Power Fx commands in the model-driven command bar cannot invoke cloud flows directly. This is consistent with the note from Phase 3 (the Power Apps V2 trigger does not surface in the command bar the way the legacy trigger did), and it is corroborated by Microsoft community and MVP sources. Three workarounds were evaluated: a JavaScript web resource command, a custom-page wrapper using a canvas `flowName.Run()` call, and deferral.

The decision was to defer to post-certification. The underlying closure approval flow is fully functional and can be invoked from the Power Automate test panel; the button is user-experience convenience over working functionality, not core capability. The planned approach for the revisit is the custom-page wrapper, since canvas pages can call V2 flows directly. Documenting the limitation and making a reasoned, timeboxed deferral was a deliberate engineering-judgment decision, not an incomplete task.

---

## Section 5: Personas and architecture diagram

Two documents were produced. The personas document defines the roles the solution serves (Agent, Requester, and the Manager/Approver sub-role) as user stories, with each story mapped to the capability that fulfills it, and with the seeded test identities and developer identities documented transparently. The architecture diagram is a high-level Mermaid diagram, rendering natively in GitHub, that shows the users, the four surfaces, the identity layer, the automation layer, and the shared Dataverse model, and how they relate.

## Section 6: README

The portfolio README replaced the Phase 0 placeholder. It opens with the project's purpose and the certification context (stated plainly, including that Arcova Advisory is fictional), lists the technology stack, describes the four surfaces, walks the phases with links to each phase document, presents three technical highlights in depth, and includes an honest disclosure of how AI was used as a collaborative tool in the project. Screenshots are placed contextually next to the prose they illustrate rather than collected in a single gallery.

---

## Lessons learned

- **The ChatControl requires a non-empty `agentTitle`.** Leaving it blank causes the control to render in a continuous loop that steals input focus. Setting any non-empty string resolves it.
- **Empty string properties are not empty.** Several control properties defaulted to two literal quote characters rather than a genuinely blank value, which can cause unintended repeated behavior. They must be cleared to truly empty.
- **Adaptive Cards render with higher fidelity in the WebChat embed than in the authoring pane.** Container styles that look subtle in the Copilot Studio test pane appear bold and correct in the embed.
- **The two dev modes are independent.** The Canvas App and the Copilot agent each have their own dev-mode toggle. In the embed, the agent resolves identity via single sign-on (real Microsoft 365 identity) while the portal in dev mode uses its persona switcher; they converge on the same identity by different paths. The production configuration is single sign-on on both.
- **Platform documentation is directional.** Across the Copilot and embed work, the reliable approach was to verify behavior against the actual product rather than assume the documentation was current, especially given the deprecation of the built-in control and the preview status of its successor.
- **A documented deferral is a stronger artifact than a forced feature.** The Closure Approval button was deferred on documented platform grounds, which demonstrates current-platform awareness and disciplined scoping.

---

## AB-410 exam connection

Phase 5 maps onto several AB-410 (Intelligent Applications Builder Associate) skill areas:

- **Integrating agents with applications.** The core of the phase, embedding a Copilot Studio agent into a Power Apps canvas app with authenticated, identity-aware access, is a direct exercise of agent-integration skills.
- **Configuring authentication and security for agents.** The Entra app registration, the single sign-on configuration, the delegated Copilot invoke permission, and the account-scoped access that carries through the embed all exercise the security and identity portions of the exam.
- **Surfacing agent capabilities across channels.** Verifying that the agent's authored topics behave consistently across the test pane and the embedded surface reflects the multi-channel surfacing skill area.
- **Testing and validating intelligent solutions.** The focused integration test suite, and the reasoning about what to test versus what was already validated, reflect the testing and validation expectations.
- **Engineering judgment under platform constraints.** The embed-mechanism decision (navigating a deprecation and a preview successor) and the documented deferral of the closure button reflect the kind of current-platform decision-making the certification is meant to validate.

---

## Phase close

With Phase 5 complete, the Arcova Engage solution is finished for the pre-certification milestone. All four surfaces are built, integrated, and verified coherent; the automation backbone is in place; the Copilot agent is embedded with single sign-on; and the portfolio documentation set is complete.

Two items are carried to post-certification: the Closure Approval command bar button (custom-page approach) and a possible Phase 6 of Power BI reporting enhancements. Neither blocks the project as a portfolio piece or as preparation for the AB-410 exam.
