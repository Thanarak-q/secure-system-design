---
name: system-design-threat-model
description: Design backend systems or review existing services with HLDs, STRIDE/LINDDUN threat models, implementation specifications, and remediation plans. Use for architecture reviews, threat modeling, and deep service-by-service reviews of all services, including cross-service attack paths and human-readable HTML reports.
---

# System Design with Threat Modeling

Produce three linked artifacts for a backend feature:

1. **HLD** — what we are building and why
2. **Threat model** — what can go wrong, what we will do about it
3. **Deep Dive** — function-level specs an engineer can implement from

The value is in the links between them. Threat modeling finds design flaws that get folded back into the HLD, and each Deep Dive line traces to a threat it mitigates. A threat model that changes nothing has failed, no matter how many threats it lists.

## Review modes

**Designing something new** — Stages 1 to 7 below.

**Working on something that already runs** — read `references/existing-system.md` instead. The flow inverts: reconstruct the diagram from the running system, verify threats rather than predicting them, and produce a remediation backlog instead of a spec. Use this mode whenever the system is deployed, whenever the user shares existing code or config, or whenever they ask to review, audit, or fix something rather than design it.

**Reviewing all services** — read `references/service-review.md`, then use the existing-system and threat-modeling references for each service. Inventory every in-scope service, review its flows deeply, then reconcile cross-service attack paths. Prioritization determines order, not permission to drop services. Produce an HTML report alongside the engineering Markdown by default in this mode.

For a bounded review, scope the requested flows first. For an all-services review, keep a coverage ledger and work in manageable batches until every service is reviewed or explicitly blocked.

The difference matters. In an existing system you can send the request and see what happens, so assumptions become checks — and a confirmed finding carries entirely different weight from a predicted one.

A mixed case is common: adding a feature to a running system. Design the new feature with Stages 1 to 7, but treat everything it touches as existing — verify the surrounding behaviour rather than assuming it works as documented.

## Where the user is

Ask or infer, then jump in at the right stage. Do not restart from requirements if they already have an HLD.

| They have | Start at |
|---|---|
| A vague idea | Stage 1 |
| Requirements | Stage 2 |
| Entities but no API | Stage 2b |
| An HLD or architecture diagram | Stage 3 |
| A DFD but no threats | Stage 4 |
| A threat list, all Open | Stage 5 |
| Threats with mitigations | Stage 6 |
| Everything | Stage 7 |
| All services or a multi-service deep review | `references/service-review.md` |
| A running system | `references/existing-system.md` |

State which stage you are entering and why, in one line. Then work.

## It is a loop, not a pipeline

Every stage can send you back to any earlier one, including all the way to requirements.

```
req → entities → api → HLD → threat model → deep dive
 ↑                                              │
 └──────────────────────────────────────────────┘
```

A Deep Dive detail can invalidate a requirement. Discovering that a per-credential limit lets a user multiply their allowance by creating more credentials is a Deep Dive finding, but the fix changes what the requirement should have said about limits in the first place.

Going backwards is the process working, not a mistake. What is a mistake is fixing the detail and leaving the earlier document saying something that is now false — the next reader follows the stale version.

So when a later stage changes an earlier decision:

1. Fix the earlier artifact, not just the current one
2. Note what changed and why
3. Check whether anything else downstream depended on the old version

## Stage 1 — Requirements

Extract functional requirements as a numbered list, and non-functional ones separately (scale, latency, availability, cost).

Pull concrete numbers from the user: how many users, how many concurrent, what budget. These numbers decide rate limits and capacity later. If they do not know, note it as an open question rather than inventing a figure — a made-up number that propagates into a rate limit config is worse than a blank.

## Stage 2 — Entities, then API, then back to entities

Three steps, in this order:

**2a. Entities from the requirements.** Pull the nouns out. Names and the fields that are obvious from the requirements alone.

**2b. Design the API.** Endpoints, methods, request and response shapes for every requirement.

**2c. Return to the entities and fill in what the API needs.** The API reveals fields the requirements did not mention — what has to be stored to serve each response, what has to be accepted from each request.

Step 2c is the one people skip, and it is where the data model stops being a guess. It also tends to shrink the entity list for a feature extending an existing system: the user entity may only need an ID for reference, because everything else about users already lives elsewhere. That becomes clear from the API and does not from the requirements.

Expect the entities to grow once more after the HLD, when cross-cutting concerns add fields — a status column, an audit timestamp, a last-used marker.

When the user gives a reason for a design choice, check whether the reason actually holds. Reasons that sound sensible but do not survive scrutiny are the most valuable thing to catch this early — see `references/hld.md` for worked examples.

## Stage 3 — High-Level Design

Read `references/hld.md`.

Produce a component diagram and a walkthrough of each requirement's flow. Add cross-cutting concerns one at a time (auth, rate limiting, quota, caching), reviewing the flow after each addition.

Record every decision with its reason. "We chose X" is not enough — "we chose X over Y because Z" is what lets someone revisit it later when Z stops being true.

## Stage 4 — Threat model

Read `references/threat-modeling.md`. This is the longest stage and where the most value is created.

Four questions, in order:

1. **What are we building?** — Build a DFD from the HLD
2. **What can go wrong?** — STRIDE per element, optionally LINDDUN for privacy
3. **What are we going to do about it?** — Mitigation per threat, then a status decision
4. **Did we do a good job?** — Review for gaps

Two failure modes to avoid. The first is a single undifferentiated backend process, which makes every threat too vague to act on. The second is stopping at "Open" for every threat, which means no decision was actually made. Both are covered in the reference.

## Stage 5 — Refine the HLD

Split mitigations into two piles:

**Pile A — changes the architecture.** New components, new flows, new trust boundaries. These go back into the HLD, producing v2.

**Pile B — implementation detail.** Query shapes, timeouts, header values. These become Deep Dive requirements.

Pile A is the proof that threat modeling worked. Name the specific changes it produced — that is what makes the difference between a document and a process.

## Stage 6 — Deep Dive

Read `references/deep-dive.md`.

For each process in the DFD, write a function spec: signature, ordered steps, failure modes, and the constants it depends on. Also specify every data store key and schema.

Two principles shape most of the good decisions here:

**Order steps cheap-to-expensive.** Anything that can reject a request without needing a rollback goes first. A validation that runs after a reservation forces you to release the reservation.

**Prefer structure over runtime checks.** If a function takes a token that can only be obtained by passing an earlier check, no code path can skip that check. If it takes a plain string, every future code path is a chance to forget. This is a stronger guarantee than re-validating, and it costs nothing at runtime.

Both are expanded in the reference with examples.

## Stage 7 — Test plan

Read `references/test-plan.md`.

Turn threats into test cases, split by who can run them: automated tests owned by the developer, manual tests owned by whoever tests, and checks that belong to another team entirely.

The highest-value tests are the ones that span multiple functions. Anyone testing function-by-function will miss them, so call them out separately.

## Output format

For a bounded review, default to a single markdown file with all stages. For all-services reviews or an explicit HTML request, follow `references/service-review.md` for the linked Markdown and offline HTML deliverables. Otherwise, adapt the output when the user has a tool they are already working in (Threat Dragon, Excalidraw, a wiki). Match their tool if they have one.

For Excalidraw, emit clipboard JSON they can paste — one text element per function spec, laid out in columns. Never silently drop elements they already had; ask for the current contents or add only new elements.

## Keeping it honest

**Do not lower a severity to make a report look better.** If something costs the organisation real money or exposes another user's data, it is High or Critical regardless of how inconvenient that is.

**"Accepted" with a reason is a real outcome.** Some threats cannot be closed — a gap between a commit and a network response, a timing difference inherent to caching. Recording the decision and its reasoning is better than inventing a mitigation that does not work.

**Flag your own earlier mistakes.** When a later stage reveals that something written earlier was wrong, say so plainly and correct it. A mitigation that says "key by X" when the design later moved to Y will mislead whoever reads it next.

**Track what you do not know.** Maintain a running list of questions for other teams, grouped by who to ask. Many threats cannot be closed without an answer, and one question often unblocks several threats at once — so getting answers is usually higher leverage than writing more specs.
