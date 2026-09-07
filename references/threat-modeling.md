# Threat Modeling

Four questions: what are we building, what can go wrong, what will we do, did we do a good job.

---

## Question 1 — What are we building?

Turn the HLD into a Data Flow Diagram. The HLD says what the system does; the DFD says what data moves where and which parts you control.

### Elements

| Element | Shape | Is |
|---|---|---|
| External entity | Rectangle | A person or system outside your control |
| Process | Circle | Something that acts and decides |
| Data store | Two parallel lines | Somewhere data rests |
| Data flow | Arrow | Data in motion — labelled with what is moving |
| Trust boundary | Dashed line | Where the level of trust changes |

A process **does something**. A flow **carries something**. Getting this wrong is the most common early mistake: putting "Authenticate session" on an arrow rather than in a circle means there is nowhere to ask "can this check be bypassed?", and that question is where the real threat lives.

### Decompose the processes

An HLD legitimately shows one backend box. A DFD must not — one box produces threats like "the backend could be bypassed", which nobody can act on.

Decompose by responsibility, not by deployment. A monolith still gets multiple processes in its DFD.

Split when any of these hold:

1. Different privilege levels
2. Touches a different data store
3. Makes a security decision that would hurt if skipped
4. Fails differently from its neighbours

The point of splitting is that it lets you ask about **order and bypass**:

- Does anything expensive run before the first control?
- Can any path reach the last step without passing the middle ones?

Neither question is askable when it is one box. This is where controls like an unauthenticated-traffic rate limit get discovered — a per-user limit cannot run until you know which user, so something has to guard the gap before it.

### Trust boundaries

More than one. A single boundary hides most of what matters. Typical set:

- Browser ↔ backend (client code you do not control)
- Untrusted client ↔ backend (no browser protections at all)
- User zone ↔ admin zone (different privilege reaching the same system)
- Backend ↔ third party (data leaving the organisation)
- Your state ↔ another team's state (schema you do not own)

Flows crossing a boundary are where threats concentrate. Get containment right before generating threats — tools compute crossing flows from containment, so a misplaced element produces a wrong threat list.

### One diagram per flow

Do not draw everything at once. Separate diagrams for the main request path, the management path, and the admin path. Each stays readable; together they cover the system.

---

## Question 2 — What can go wrong?

### STRIDE per element

Element type determines which threats apply:

| Element | S | T | R | I | D | E |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| External entity | ✓ | | ✓ | | | |
| Process | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Data flow | | ✓ | | ✓ | ✓ | |
| Data store | | ✓ | ✓* | ✓ | ✓ | |

\* Repudiation applies to stores that are meant to be evidence.

Flows and stores have no identity to impersonate and no privilege to escalate, which is why S and E do not apply to them.

### Scope the work

Full coverage produces roughly `processes × 6 + flows × 3 + stores × 4` threats, which is more than most projects can act on. Prioritise:

1. Flows crossing trust boundaries
2. Processes making security decisions
3. Stores holding secrets or shared state

Record what you deliberately skipped and why. Silence reads as oversight.

### Questions that generate real threats

Generic tool output ("an attacker could tamper with data in transit") applies to every system and helps with none. These questions produce specific threats:

**Inbound flow** — what in this request does the caller control, and what happens before the first control runs?

**Outbound flow with anything sensitive** — trace the copies: application log → proxy log → APM → error tracker → cache → client storage. Each hop is a place it can persist.

**Any process reading a cache** — what is the window between the source of truth changing and the cache reflecting it, and what if invalidation fails?

**Any process depending on infrastructure** — when it is unavailable, does the system fail open or closed? If the answer is not written down, that is the threat.

**Any multi-step state change** — is it atomic, or can concurrent requests interleave?

**Any control that runs before the value is used** — can a path reach the use without passing the control?

**Anything shared with another team** — what happens when they change it without telling you?

### Threats that recur across systems

- Controls that run after the resource is consumed
- Rate limits keyed on a value the caller can rotate
- Per-key limits that multiply when a user creates several keys
- Read-then-write where atomic is required
- Secrets reaching logs through exception traces, which redaction config does not cover
- Cache invalidation failure leaving revoked credentials live
- Resource reservations with no expiry, accumulating permanently
- Ownership checked in code after a fetch rather than in the query itself
- Client-side restrictions treated as controls
- Values trusted because an earlier step was supposed to have validated them

### Deduplicate by mitigation

Per-element analysis surfaces the same underlying issue at several elements. Group them:

```
Transport security       → 3 threats, one control
Secrets in logs          → 4 threats, one config change
Failure mode undefined   → 6 threats, one decision
```

Six threats behind one decision is one piece of work, not six. Grouping shows the real size of the remaining effort, which matters when someone is deciding whether the work is feasible.

---

## LINDDUN (privacy)

Worth adding when the system handles personal data, sends data to a third party, or logs user behaviour.

| | Ask |
|---|---|
| **L**inking | Can two records be tied to the same person? |
| **I**dentifying | Can a person be identified from this? |
| **N**on-repudiation | Can they be forced to admit something they should be able to deny? |
| **D**etecting | Can you tell whether someone exists in the system? |
| **D**ata disclosure | Does data reach someone who should not have it? |
| **U**nawareness | Do users know what happens to their data? |
| **N**on-compliance | Does this meet the applicable regulation? |

Skip categories that do not apply. In an authenticated internal system, Identifying and Detecting are usually moot — identity is the point.

Data disclosure overlaps heavily with STRIDE's Information disclosure. Do not redo it; reference the STRIDE work.

### STRIDE and LINDDUN conflict

STRIDE's Repudiation wants stronger attribution. LINDDUN's Non-repudiation wants users able to deny. Both are legitimate.

Do not resolve this by picking one silently. Record the conflict, decide, and say why:

> Usage records are retained for 60 days to resolve billing disputes. Source IP
> is captured for credential operations and authentication failures only, not
> for successful requests, since a per-request location trail is not needed for
> that purpose and would constitute behavioural tracking.

Naming the tension and resolving it deliberately is stronger evidence of thinking than either framework's output alone.

---

## Question 3 — What are we going to do?

Every threat needs a mitigation and a status. A list of threats all marked Open means no decision has been made.

| Status | Means |
|---|---|
| Mitigated | Control designed and specified |
| Accepted | Understood, deliberately not addressed, reason recorded |
| Transferred | Another team owns it |
| Out of scope | Outside this feature's boundary |
| Open | Genuinely undecided, usually blocked on an answer |

### Writing mitigations

Specific enough to implement. "Validate input" is not a mitigation; "reject requests where the model name is not an exact match in the allowlist, before building the provider request" is.

Include the reason when the choice is not obvious, especially where a reasonable engineer might "improve" it into something worse:

> Hash with SHA-256, not bcrypt or argon2. The secret has high entropy from a
> CSPRNG, so the slow-hash property that protects low-entropy passwords buys
> nothing here, and this runs on every request.

Without that note, someone will eventually swap it for bcrypt believing they are hardening it.

### Accepted is a real answer

Some threats cannot be closed:

- A gap between a database commit and a network response
- Timing differences inherent to caching
- What a third party does with data after it arrives
- What downstream consumers do with your output

Record them honestly:

> Accepted. Full recovery is impossible by design — the plaintext is never
> stored. This residual risk is accepted in exchange for the guarantee that a
> database compromise yields no usable credentials.

That is a stronger position than a mitigation that does not work.

### Severity

Judge by blast radius and cost, not by how hard it is to fix.

- **Critical** — one user reaches another user's data or actions
- **High** — real money, credentials, or system-wide availability
- **Medium** — limited-scope disclosure or degradation
- **Low** — information leak of little value

A shared organisational credential is more severe than an individual one — one leak affects everybody.

---

## Question 4 — Did we do a good job?

Check for:

- Elements with no threats — deliberate, or skipped?
- Threats still Open with no reason
- Mitigations copied from the description field
- Threat types that do not match the element type
- Mitigations contradicting the current design (common after the design evolved)
- Placeholder text left in
- Flows in the diagram with no threats at all

### Contradictions after the design evolves

The design will change during Deep Dive, and mitigations written earlier can go stale. When a mitigation says "key the bucket by key ID" but the design moved to user ID, fix the threat model — not just the code. Otherwise the next reader follows the stale instruction.

### It repeats

Threat modeling is not a phase. Redo it when:

- Architecture changes (which the mitigations themselves will cause)
- New components appear
- Trust boundaries move
- A new interaction pattern is supported

Note in the document when it should next be revisited.
