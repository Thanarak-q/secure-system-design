# High-Level Design

## What goes in

- Component diagram — boxes and arrows, no internal detail
- One walkthrough per functional requirement
- Data stores and what each holds
- External dependencies
- Every decision with its reason
- Assumptions and constraints
- Open questions

## Building it up

Do not try to draw the final diagram in one pass. Start with the minimum that satisfies the functional requirements, then add cross-cutting concerns one at a time, reviewing the whole flow after each:

```
1. Base flow satisfying the requirements
2. + authentication
3. + rate limiting
4. + quota / cost control
5. + caching
6. + admin management (if anything is configurable at runtime)
```

Each addition tends to reveal something the previous step missed. Adding model validation, for instance, implies someone must be able to change which models are allowed — which means an admin flow you had not drawn.

## Recording decisions

The reason matters more than the choice. Write:

> Chose token bucket over fixed window because usage is bursty — users fire several requests then go quiet. Fixed window has boundary spikes; sliding log costs too much memory at this scale.

Not:

> Using token bucket.

Without the reason, the next person cannot tell whether the decision still applies when the situation changes.

## Test every stated reason

The most valuable thing to catch at HLD stage is a reason that sounds right but does not hold. These are hard to spot later because they read as already-settled.

**Example 1 — the reason protects the wrong thing**

> "We use POST with the ID in the request body instead of the URL, to avoid exposing it in access logs and browser history."

Sensible-sounding, but the ID is not secret — it is displayed in the UI. The actual risk is operating on a resource you do not own, and moving the ID out of the URL does nothing about that. The real control is scoping the query by owner.

Keeping POST is fine. The reason needs to change, and the real control needs to be identified.

**Example 2 — the check happens after the cost is incurred**

> "The provider enforces the maximum token limit, so we do not need to."

The provider enforces it after processing the request, which is after the bill is generated. A limit that protects your spending has to run on your side.

**Example 3 — the reason is authority, not analysis**

> "Using this algorithm because a well-known system uses it."

May land on the right answer, but does not survive review. Find the property that makes it right for this system.

## Assumptions

State them explicitly, especially about things you cannot verify:

```
- The LLM provider is treated as a generic third party with no organisational
  agreement. Where an agreement exists, related threats can be downgraded.
- The application runs as a monolith; the feature shares worker pool and
  database with the main application.
- Quota state is owned by the main application team.
```

Assumptions are what let a reviewer tell "considered and set aside" apart from "not thought about". They also tell you which threats to revisit when circumstances change.

## Open questions

Keep a live list grouped by who can answer:

```
Platform team
  □ Does the proxy overwrite X-Forwarded-Proto?
  □ Redis version and configuration?

Network team
  □ Does user traffic egress through NAT (one shared IP)?

Owning team for shared state
  □ How is the shared counter updated — atomic increment or read-modify-write?

Verify directly
  □ Is there an APM or error tracker capturing request context?
```

Answers unblock threats. Track which threats depend on which question — one answer often closes several, which makes chasing answers higher leverage than writing more spec.
