# Deep reviews across services and HTML reporting

Use this workflow for an explicit all-services review. Use the reporting section alone when a bounded review requests HTML. Read `existing-system.md` for evidence and remediation guidance and `threat-modeling.md` for DFD and threat analysis.

## 1. Inventory and preserve scope

Discover deployable services from manifests, routing, code, workers, scheduled jobs, and infrastructure configuration. Record shared identity providers, stores, queues, gateways, observability, and external dependencies. A monolith can have logical service areas; explain the decomposition instead of inventing deployments.

Create stable service IDs and a coverage ledger before deep analysis. Service IDs prefix the element and threat IDs defined in `threat-modeling.md`, so fix them here and do not renumber them later:

| Service ID | Paths / revision | Entry points and flows | Dependencies | Review state | Evidence gaps / next step |
| --- | --- | --- | --- | --- | --- |

Use `pending`, `in progress`, `reviewed`, or `blocked`. An inventory entry or diagram alone is not a completed review. Missing source for an external dependency means an interface-level review, with its internals explicitly unexamined.

Prioritize exposure and impact to choose the order. An all-services request retains every discovered in-scope service. Review sequentially by default; do not require multi-agent tools. If work must pause, save the ledger, current findings, and exact next flow. Report partial coverage honestly and resume from that checkpoint.

Record the repository revision and available deployment evidence. Code on a branch is not proof of production behavior. When new dependencies appear during analysis, add them to the inventory and affected flows.

## 2. Deep review of each service

Complete the following for one service before moving to the next, except where missing evidence blocks it:

1. Establish its purpose, assets, callers, data sensitivity, privilege, tenant model, and security invariants.
2. Enumerate HTTP/RPC handlers, queue consumers, scheduled jobs, admin/debug routes, and ingress overrides from actual registration and configuration. List each relevant flow and its review state.
3. Trace flows from input through authentication, authorization, parsing, business decisions, storage, downstream calls, and responses or side effects. Include failure paths, retries, idempotency, races, resource limits, secret handling, and log/error copies where relevant. Inspect the shared helpers and other callers that determine whether a control can be bypassed.
4. Build detailed DFDs for distinct flows. Show external entities, security-relevant processes, stores, labeled directional flows, and trust boundaries. Include protocol, identity, and data carried across boundaries. Decompose a service's security decisions rather than drawing only one opaque box.
5. Apply relevant STRIDE categories per element and optional privacy analysis. Record considered categories with no supported threat as reviewed, not as invented findings. Trace concrete attack paths and existing controls with source/configuration references or observed evidence.
6. For each supported threat, write the full record from `threat-modeling.md` — ID, element and flow IDs, category, prerequisites, attack steps, asset, impact, severity rationale, evidence and confidence, current controls, proposed mitigation, owner (or unknown), and a verification test. Qualify the element IDs with the service ID so they stay unique across the review. Keep evidence state separate from remediation status.
7. Produce the service's prioritized remediation work, compatibility risks, residual risk, and unanswered questions. A recommendation alone does not close a finding. Acceptance requires an actual recorded owner decision; otherwise mark acceptance as proposed.

`reviewed` means enumerated flows were traced to their consequential operations, relevant threats assessed, and evidence gaps documented. Any missing material that prevents that trace leaves the service `blocked` or `in progress`. It does not mean vulnerability-free.

Do not impose a threat count or skip low-exposure services simply because high-exposure services produced enough findings. Continue independently reviewable work when a service is blocked.

## 3. Cross-service review

Service ownership organizes the work; attack paths still cross service boundaries. After the individual reviews, trace each connecting flow from sender to receiver and reconcile both accounts of it:

- Propagation of caller identity, tenant identity, authorization, and delegated privileges.
- Gateway enforcement versus direct/internal access and worker entry points.
- Queue provenance, replay, retries, dead-letter handling, and idempotency.
- Shared stores, caches, credentials, revocation, and consistency assumptions.
- Failure propagation, resource exhaustion, third-party egress, and sensitive observability data.

Create a system overview DFD that links to the service diagrams. Track cross-service flows in the ledger as well as services; full service coverage with unreviewed connections is incomplete.

Deduplicate shared root causes into one remediation item while preserving every affected service, threat ID, and attack path. Do not merge distinct vulnerabilities merely because their fixes both say “authorization.” Update earlier service conclusions if the combined flow contradicts them.

## 4. Human-readable deliverables

For an all-services review, generate engineering Markdown and a self-contained `report.html`. For a bounded HTML request, include only the requested scope. Generate both views from the threat records described in `SKILL.md` — `security-review/threats.yaml` — so IDs, counts, severity, and status agree by construction rather than by proofreading. Keep detailed evidence in Markdown or linked report sections; never maintain two findings lists.

The HTML must open directly from disk without a server or internet connection. Use embedded CSS, native anchors, tables, and `<details>` disclosures. Render diagrams as inline SVG, or embed locally rendered images; raw Mermaid text without a renderer does not satisfy the visual requirement. Add only small inline JavaScript when search or filtering materially helps a large report. Keep full content readable with JavaScript disabled.

Include:

- Scope, revision/date, coverage counts and denominator, limitations, and confirmed versus unverified findings clearly separated.
- A service index with review states and links to each service section.
- A system DFD and detailed service/flow DFDs with readable labels and a legend.
- Threat entries linked from diagram element/flow IDs, including evidence, severity rationale, remediation status, and verification work.
- Cross-service attack paths and a deduplicated remediation backlog.
- Open questions, blocked work, and residual risks. If no threats were supported, show coverage and limits rather than a blank report or an “all secure” badge.

Use semantic headings, keyboard-accessible links/disclosures, visible focus, sufficient contrast, and print styles. Never rely only on color to convey severity. Escape source snippets, labels, and findings as text; never insert untrusted content as raw HTML or executable script. Exclude credential values and unnecessary personal data. Do not fetch remote fonts, scripts, or diagram services with confidential architecture data.

Before delivery, verify unique IDs, valid diagram/threat links, coverage totals, and matching threat counts across views — against the records, not by eye. Include the coverage matrix from `threat-modeling.md` per service, so a reader can tell an element that was cleared from one that was skipped. Open the HTML in an available browser and check diagrams, navigation, narrow-screen readability, and printing. If a browser is unavailable, run structural checks and explicitly report visual verification as pending. Do not claim an HTML report exists until its file has been generated.

## 5. Threat Dragon interoperability

A Threat Dragon-style review means DFD elements connected to threats, mitigation decisions, and evidence. An HTML report is a human view, not an importable Threat Dragon file or a replacement diagram editor.

When the user explicitly requests Threat Dragon export or supplies a model, preserve their model and IDs. Inspect the target version's official schema and a valid example before producing JSON; do not invent a compatible schema. Validate exported JSON against that schema and, when the app is available, import and reopen it to check diagram layout, containment, flows, and attached threats. State exactly which checks ran. An untested export must be labeled unverified. A blocked export does not block the Markdown and HTML review.

Official starting points: [Threat Dragon documentation](https://www.threatdragon.com/docs/) and its Schema and Modeling sections.
