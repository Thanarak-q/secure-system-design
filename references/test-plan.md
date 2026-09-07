# Test Plan

Turn threats into tests, and be explicit about who runs each one. A test plan that mixes "check the log file" with "click the button" gets partially abandoned by whoever receives it.

## Who can run what

| Test | Needs | Owner |
|---|---|---|
| Ordering of internal steps | Code access | Developer |
| Atomicity under concurrency | Concurrent runner + often code access | Developer |
| Secrets absent from logs | Log access | Developer |
| Cache TTL behaviour | Timing plus internal knowledge | Developer |
| Infrastructure config | Server access | Platform team |
| Cross-account access | Two accounts + request editing | Tester |
| Response consistency | Request comparison | Tester |
| Client-side storage | Browser dev tools | Tester |
| Flow behaviour | Normal access | Tester |

Tell the tester explicitly what is not theirs. Otherwise they write "unable to test" against items that were never in their scope, which reads as incomplete work.

## Automate the severe ones

Threats that are Critical or High and objectively checkable belong in CI, not in a manual pass:

```
- cross-account access on every mutating endpoint
- concurrency against a shared counter
- limit scope when one user holds several credentials
- transport enforcement including a forged proxy header
- secret absence in logs, including after a thrown exception
```

The exception case matters most — it is the path that redaction configuration does not cover, so a grep of normal logs will pass while the real leak sits in the error tracker.

Tag each test with the threat it covers:

```
test_reject_forged_proxy_header()
  // covers: transport enforcement
  // failing here means the proxy is not overwriting the header,
  // which makes the application-level check meaningless
```

This lets you answer "which threats have test coverage" directly, which is the question a reviewer actually asks.

## Cross-function tests

The highest-value tests span several functions. Anyone working function-by-function will not see them, so list them separately and mark them as first priority.

```
1. Revoke, then use immediately — measure how long until it stops working,
   compare against the documented exposure window
2. Two credentials on one account — do they share a limit or multiply it
3. Heavy use here — does the shared allowance decrease in the other system
4. Account B acts on account A's resource — correct rejection, and A unaffected
5. Concurrent requests against a shared counter — does the total hold
```

These map directly to the threats that are hardest to reason about statically, which is exactly why they need runtime verification.

## Test case format for handoff

```
TC-ID  Name                                    [Priority]

Why: one line — what breaks if this fails

Steps:
  concrete commands or clicks, runnable without interpretation

Expected:
  specific status, body, and behaviour

Fail if:
  the specific wrong outcomes, and which ones to report immediately
```

State the "why" in one line. Without it a tester cannot tell which failures are urgent, and everything gets the same weight in the report.

## Verify controls that depend on infrastructure

Some application-level controls only work if something upstream behaves correctly — reading a header the proxy is supposed to overwrite, for example. Test both halves:

```
- application rejects the bad value        (unit test)
- forged value from a client is rejected   (integration, hits the real stack)
```

The second is the one that matters. Passing the first while failing the second means correct code and a broken system. Run it against the real deployment on every release, because infrastructure config changes without the code changing.

## Report format

Ask for:

```
TC-ID   PASS / FAIL
  actual:       what happened
  reproducible: always / intermittent / once
  evidence:     output, screenshot
  notes:        variations tried
```

Reproducibility changes the diagnosis. An intermittent failure on a concurrency test is a race condition; a consistent one is straightforward logic.

Name which failures warrant an immediate report rather than waiting for the round to finish.
