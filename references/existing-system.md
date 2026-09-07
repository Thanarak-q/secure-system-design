# Existing Systems

Threat modeling something already built differs from designing something new in three ways that change the whole approach.

**You can verify.** In design work, "the proxy probably overwrites that header" is an assumption. Here you can send the request and find out. Every assumption you would have written down is instead a five-minute check, and checks beat assumptions.

**Reality diverges from documentation.** The diagram shows what someone intended. The system does what it does. Endpoints exist that nobody wrote down, a control was disabled during an incident and never re-enabled, a config drifted. The DFD must describe the running system, not the documented one.

**Change carries risk.** A greenfield fix costs implementation time. A fix here can break existing callers, so the remediation plan needs to weigh blast radius alongside severity.

---

## Stage 0 — Scope, if the system is large

Anything beyond a single feature needs scoping first. A system with fifty endpoints yields thousands of threats under full per-element analysis — nobody reads that, and nothing gets fixed. Completeness is not achievable here, and chasing it produces nothing.

Two passes: broad and shallow to find where to look, then narrow and deep on what you found.

### Pass 1 — map, do not analyse

The output is a map, not a threat list. No STRIDE yet. Time-box it to a day or two; going longer means you have started analysing.

```
□ Every entry point taking traffic from outside
□ Where the valuable things are
   credentials, personal data, money, admin privilege
□ Coarse trust boundaries
□ Which flows cross them
```

### Choose two or three slices

Rank by exposure and blast radius, not by how the code is organised:

| Signal | Why |
|---|---|
| Reachable from the internet without authentication | anyone can reach it |
| Handles credentials or money | large blast radius |
| Shares state with another system | failure crosses team lines |
| Has a privilege escalation path | admin surface |
| Changed recently | new code carries new defects |

Pick two or three. Not more — the point of scoping is defeated by scoping widely.

### A slice is a flow, not a module

```
Good:  the authenticated file upload path, entry to storage
Bad:   the storage service
```

Threats live in how data moves, not inside boxes. Slicing by module cuts flows in half and hides exactly the boundary crossings you are looking for.

A workable slice has one clear external entry point, one clear endpoint, crosses at least one trust boundary, and contains five to eight processes. More than that and it should be two slices.

### Pass 2 — full analysis, per slice

Run Stages A to C below on each chosen slice. One slice typically yields twenty to thirty threats, which is a volume a team can actually work through.

### Record what you did not examine

This carries as much weight as the analysis itself:

```
Analysed:      authenticated upload path, credential issuance
Not analysed:  reporting module, batch import, internal console
Reason:        no external entry point, no credential handling
Revisit when:  any of them gains an external interface
```

Without this, a reader assumes the whole system was covered. An unstated gap is more dangerous than a stated one, because it removes the prompt to go look.

### Scoping failures to avoid

**Trying to be complete.** You cannot be, and the attempt consumes the time that would have produced a few real fixes.

**Slicing by module.** Produces scattered findings and hides real attack paths.

**Filling volume with theoretical threats.** Large systems offer endless material for category-level observations. Without forcing yourself to reproduce the severe ones, you get a thick document nobody acts on.

**Leaving the boundary unstated.** Readers conclude the system was cleared.

---

## Stage A — Reconstruct the DFD from reality

Build the diagram from evidence, not from documents. Documents are a starting hypothesis to confirm.

If you scoped in Stage 0, do this per slice rather than for the whole system.

### Sources, in order of trustworthiness

1. **Running behaviour** — what responds, what headers come back, what a request actually does
2. **Configuration** — routing, proxy rules, environment, infrastructure-as-code
3. **Code** — route definitions, middleware registration, database queries
4. **Documentation** — the least reliable; treat as a claim to test

### Enumerate what actually exists

The gap between the documented surface and the real one is often where the findings are.

```
□ Every route the router actually registers
   compare against documented endpoints — note extras
□ Which routes bypass the auth middleware
   look at registration order and any per-route overrides
□ Debug, health, admin, and internal endpoints
   these are the ones that get forgotten
□ Every data store the application connects to
□ Every outbound destination
□ What observability captures — APM, error tracking, log shipping
   these are components in the DFD, and they usually are not in the diagram
```

Observability tooling is worth calling out. It touches sensitive data, nobody draws it, and it captures request context through a path that logging configuration does not cover.

### Note the drift

Record where reality and documentation disagree. Each gap is either a documentation bug or a real finding, and you cannot tell which until you look:

```
Documented: all routes behind authentication
Actual:     /internal/reindex registers outside the middleware group
Status:     confirmed finding
```

---

## Stage B — Threat model, with verification

Run the same STRIDE analysis as `references/threat-modeling.md`, but with one addition: mark each threat by whether you have observed it.

| Marking | Means |
|---|---|
| **Confirmed** | Reproduced against the running system |
| **Likely** | Code or config suggests it, not yet reproduced |
| **Theoretical** | Applies by category, not investigated |

This distinction drives everything downstream. A confirmed cross-account access finding and a theoretical one are the same severity but wildly different urgency, and mixing them makes the report unactionable.

Confirm the severe ones. For anything Critical or High, spend the time to reproduce it — a confirmed finding with a reproduction is a work item, an unconfirmed one is a discussion.

### Checks that pay off quickly

These are fast and frequently productive on systems that have been running a while:

```
□ Send an unauthenticated request to every route
□ Act on another account's resource by editing an identifier
□ Compare responses for not-found versus not-permitted
□ Forge headers the application trusts from a proxy
□ Fire concurrent requests at anything with a counter or limit
□ Trigger an error and read what comes back
□ Check whether secrets appear in accessible logs
□ Look for credentials in version control history
```

The last one is worth doing early — if a credential was ever committed, deleting the file later did not remove it, and the credential needs rotating regardless of what else you find.

### Watch for controls that stopped working

Controls decay without the code changing:

- A limit whose threshold was raised for an incident and never lowered
- Verbose logging enabled for debugging and left on
- A cache TTL extended for performance, widening a revocation window
- An allowlist entry added temporarily
- A proxy rule dropped during a migration

Compare current configuration against what the design intended. Version control history on config often shows exactly when something changed and why.

---

## Stage C — Remediation plan

The output is an ordered backlog, not a design document.

### Order by severity against change risk

```
Fix now       Critical or High, confirmed, low blast radius
Fix soon      High, or Critical needing coordination
Plan          Requires architecture change or a breaking change
Accept        Documented with reasoning
```

Low blast radius first is deliberate. A query-scoping fix or a config change ships this week; an API contract change needs client migration and takes a quarter. Shipping the safe ones immediately reduces exposure while the larger work is planned.

### Note what breaks

For each fix, state the compatibility impact:

```
Fix:     scope every mutating query by owner
Breaks:  nothing — no legitimate caller relies on cross-account access
Ship:    immediately

Fix:     require an explicit resource identifier, remove the default
Breaks:  callers omitting the field currently get a default
Ship:    deprecation notice, then enforce
```

A fix that silently breaks callers gets rolled back, which leaves the system in its original state plus lost trust. Naming the impact upfront is what gets it deployed.

### Quick wins deserve their own section

Configuration changes and single-line fixes often close High-severity findings in minutes. List them separately so they are not buried behind architectural work:

```
- disable local-variable capture in the error tracker
- add the credential header to the proxy redaction list
- reduce the authorisation cache TTL
- close the debug endpoint
```

### Interim mitigations

When a proper fix is far off, a partial one now is worth recording:

```
Finding:  no rate limit on the unauthenticated path
Proper:   application-level limiting, needs a shared store — 2 weeks
Interim:  connection limit at the proxy — today
```

---

## Where fixes commonly do not belong

In an existing system, several findings will turn out to be someone else's:

- Transport and proxy configuration — platform
- Shared state semantics — the owning team
- Session handling in a system this feature extends — the host application
- Observability configuration — whoever runs it

Mark them Transferred with a specific ask, not a vague one. "Add `proxy_set_header X-Forwarded-Proto $scheme;`" gets actioned; "improve proxy security" does not.

Track them anyway. Transferred does not mean resolved, and an unanswered transfer is still an open exposure.

---

## Reporting

Two audiences, two documents.

**For engineers** — the full finding list with reproductions, ordered by the backlog above.

**For everyone else** — a page that says what was examined, what was found grouped by severity, what is being fixed and when, and what is being accepted and why. No STRIDE categories, no element names.

Both open with the scope statement from Stage 0. A reader who does not know what was left out will assume nothing was.

The second one is what determines whether the work gets prioritised. Findings nobody outside the team can read do not get scheduled.

### Do not inflate

Resist counting theoretical threats as findings. A report of six confirmed issues is more credible and more actionable than one listing eighty, most of which are category-level observations that apply to every system of this shape.
