# Deep Dive

Function-level specifications an engineer can implement from. One spec per process in the DFD, plus data store definitions, API contract, and constants.

## Function spec format

```
function_name(args) -> success | error

- ordered steps
- each step's failure mode and status code
- non-obvious reasoning inline as a sub-bullet
```

Keep it dense. An engineer should be able to work from this without re-reading the threat model, but the reasoning behind unusual choices needs to be present or it will be "corrected" later.

## Order steps cheap to expensive

Anything that can reject a request without needing a rollback goes first:

```
1. transport check          no state
2. unauthenticated limit    cheap lookup
3. authentication           cache, then database
4. per-user limit           cheap lookup
5. resource validation      cache
6. resource reservation     first step needing rollback
7. expensive external call
```

Validation placed after a reservation forces a release path on every failure. Moving it earlier removes the need entirely.

## Prefer structure over runtime checks

A control that runs before a value is used can be skipped by any path that does not pass through it. New endpoints, retry paths, background jobs, and internal calls all bypass it — none of them adversarial, all of them plausible.

Two ways to address this:

**Re-check at the point of use.** Works, costs a lookup, still relies on someone remembering to write the check.

**Make the unchecked call impossible to express.** Have the downstream function take a token that only the check can produce:

```
validate(name) -> resource_id
use(resource_id, ...)      -- no way to call without validating first
```

The second is stronger and free at runtime. Every bypass path fails to compile or fails immediately, rather than silently skipping a control.

The same pattern applies to any step whose completion later steps depend on. If a function must only run after a reservation succeeded, have it take the reservation handle.

Where the language cannot distinguish the token type from a plain integer, fall back to enforcing the chain at the router with default-deny, and note the limitation.

## Specify failure modes

For every dependency, state what happens when it is unavailable:

```
- cache unavailable -> fail closed, 503
```

Fail-open and fail-closed are both defensible, but the choice must be explicit. Fail-open on a control protecting money removes the protection at the worst moment. Fail-closed on a non-critical path causes an outage nobody needed.

Keep the choice consistent across the chain. If one component fails closed, a fallback path in a later component may be unreachable — which is either dead code or a contradiction, and both need resolving.

## Atomicity

Any read-decide-write sequence on shared state needs to be one operation. Two separate calls let concurrent requests read the same value and both proceed.

When several conditions gate one action, check them all before committing any:

```
- single script, both limits checked before either is decremented
  - if either rejects, neither is consumed
```

Otherwise a request rejected by the second check has already consumed budget from the first — repeated rejections drain a user's allowance without a single successful request.

## Do not leave dangling counters

A reservation pattern needs the reserved amount to disappear on its own if nothing cleans it up. An expiring key that records the reservation, paired with a separate counter holding the total, leaves the counter permanently inflated when the key expires unreferenced.

Store reservations as timestamped entries and derive the total by summing live ones. Expired entries are removed before each calculation, so there is no standing number to get stuck.

This also has a practical benefit: the derived total lives in your own namespace, so it does not require another team to add a field to state they own.

## Release on every failure path

Call the release in a finally/defer block, not at each error branch. There are more failure paths than are obvious — timeout, upstream error, size limit, parse exception — and missing one costs the user resources they never consumed.

Make release idempotent so retries and overlapping handlers are safe.

## Data stores

Define every key:

| Key | Type | TTL | Contents | Written by |
|---|---|---|---|---|

TTL on a cached authorisation decision is the exposure window after a revocation if invalidation fails. State it as that, not as a tuning parameter:

```
- cache ttl 60s
  - = worst-case exposure window after revocation if invalidation fails
```

Never key a cache on a raw secret. Use an identifier, and store a hash inside the value to compare against. A cache keyed by secret turns a memory dump into a credential dump.

## Constants

Collect them in one place with the reasoning:

```
MAX_ITEMS = 5              enough for separate environments, still forces cleanup
cache TTL = 60s            exposure window after revocation
limit cap = ?              blocked on: does traffic egress via NAT?
```

Mark blocked ones explicitly. A guessed capacity figure that reaches production is worse than a visible blank.

## API contract

Full request and response shapes, plus:

**A fixed error set.** Everything maps into it; anything unmappable becomes the generic server error. Never pass through a third party's error body — it can carry account state and internal identifiers.

**A correlation ID on every response.** This is what makes it possible to investigate an issue without logging request or response content. It resolves the tension between "do not log user content" and "we need to debug".

**Consistent responses for indistinguishable failures.** Not-found and not-yours must be identical in status, body, and headers — otherwise the difference confirms which identifiers are real.

**Idempotency for anything billable.** A retry after a lost response must not charge twice. Scope the idempotency record by caller, and reject a reused key carrying a different payload rather than returning the old result.

## Logging spec

An allowlist beats a denylist for fields, and both miss the same thing: exception traces capture local variables directly, bypassing logger configuration entirely.

Two habits close that gap:

```
- hash the secret at the entry point, pass only the hash inward
- keep the raw value out of any object that gets serialised
```

Log identifiers, never secrets. An identifier that is already displayed in the UI is safe to log and makes investigation possible without touching the secret.

Also verify what else captures request context — APM tools and error trackers have their own configuration and do not inherit the application logger's redaction.
