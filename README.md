# Secure System Design

An Agent Skill for secure backend design and deep service-by-service architecture reviews. It connects requirements, threat modeling, implementation planning, and human-readable HTML reports.

The skill's invocation name is **`secure-system-design`**, as defined in [SKILL.md](SKILL.md).

## What it does

- Designs new backend features, services, and APIs.
- Builds data flow diagrams and identifies trust boundaries.
- Applies STRIDE threat analysis and optional LINDDUN privacy analysis one data flow at a time, with per-element attributes, stable element IDs, and a coverage matrix that separates cleared elements from skipped ones.
- Feeds architectural mitigations back into the design.
- Produces function-level specifications and tests linked to threats.
- Reviews existing systems and produces prioritized remediation plans.
- Reviews all services one by one, tracks coverage, and traces cross-service attack paths.
- Produces an offline HTML report with linked diagrams, threats, and remediation details.

Start with what you already have: an idea, requirements, an HLD, a diagram, code, or a threat list. The workflow starts at the relevant stage.

## Workflow

<p align="center">
  <img src="assets/workflow.svg" alt="Requirements to Entities to API to HLD to Threat model to Deep Dive to Test plan, with a return loop from any stage to any earlier one." width="100%">
</p>

| Stage | Result |
| --- | --- |
| Requirements | Functional needs, constraints, and open questions |
| Entities and APIs | Data model and request/response contracts |
| High-Level Design | Components, request flows, and decisions with reasons |
| Threat model | DFD, threats, mitigations, and status decisions |
| Design refinement | Architecture updated to address threats |
| Deep Dive | Function specifications, failure modes, and storage schemas |
| Test plan | Verification linked to threats and assigned by ownership |

For an existing system, reconstruct the architecture from available evidence, assess actual behavior, and produce a remediation backlog. Distinguish predicted threats from confirmed findings and record unknowns explicitly.

A bounded review defaults to one Markdown document. An all-services review produces engineering Markdown plus an offline `report.html`, built from [assets/report-template.html](assets/report-template.html): a system design summary, then one section per data flow carrying that flow's diagram and only the threats found on it, then cross-flow attack paths, the remediation backlog, and the coverage matrix. Deliverables land in `security-review/`, and both views are generated from a single set of threat records so their IDs, counts, and statuses cannot drift apart. The workflow can also work with material from Threat Dragon, Excalidraw, or a wiki; Threat Dragon JSON is an optional export that must be validated against the target version, and HTML alone is not a Threat Dragon model.

### Reviewing all services

1. Inventory services, workers, shared infrastructure, and dependencies; record the review scope and revision.
2. Review each service's entry points and flows in depth, including security controls, failure paths, evidence, and verification tests.
3. Track each service as pending, in progress, reviewed, or blocked. Prioritization changes the order, not the requested coverage.
4. Trace cross-service attack paths and reconcile identity, authorization, shared state, queues, and failure propagation.
5. Deliver engineering Markdown and an offline HTML report with service navigation, linked DFDs and threats, and a deduplicated remediation backlog.

This repository contains instructions for the agent to generate these artifacts during a review; it does not include a standalone report application.

## Installation

Requires Node.js and npm. Pick your agent and run its line.

**Codex**

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent codex
```

Invoke with `$secure-system-design review my system architecture`

**Claude Code**

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent claude-code
```

Invoke with `/secure-system-design review my system architecture`

**Hermes Agent**

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent hermes-agent
```

Invoke with `Use secure-system-design to review my system architecture.`

**Gemini CLI**

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent gemini-cli
```

Invoke with `Use secure-system-design to review my system architecture.`

**Several agents at once** — list them after `--agent`

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent codex claude-code hermes-agent gemini-cli
```

## Example prompts

**Review all services in depth**

```text
Use secure-system-design to review all services in this repository.
Inventory them first, then deeply review each service and its flows.
Finish with cross-service attack paths and an offline HTML report.
Track any blocked or unfinished coverage explicitly.
```

**Design a new feature**

```text
Use secure-system-design to design a backend API for file uploads.
Start with requirements and identify the information you need from me.
```

**Review an existing service**

```text
Use secure-system-design to review this service's authentication
and file-access flows. Produce a prioritized remediation plan and label
unverified assumptions clearly.
```

**Continue from an HLD**

```text
Use secure-system-design with this HLD. Build the data flow diagram,
identify threats, and update the architecture with the required mitigations.
```

**Plan implementation**

```text
Use secure-system-design to turn this threat model and its mitigations
into implementation specifications and a test plan.
```

## Files and customization

```text
secure-system-design/
├── README.md
├── SKILL.md
├── agents/
│   └── openai.yaml
├── assets/
│   ├── report-template.html
│   └── workflow.svg
└── references/
    ├── hld.md
    ├── threat-modeling.md
    ├── existing-system.md
    ├── deep-dive.md
    ├── test-plan.md
    └── service-review.md
```

| File | Edit to change |
| --- | --- |
| [SKILL.md](SKILL.md) | Main workflow, activation description, and output guidance |
| [hld.md](references/hld.md) | Architecture and design decisions |
| [threat-modeling.md](references/threat-modeling.md) | DFDs, threats, and mitigation decisions |
| [existing-system.md](references/existing-system.md) | Existing-system reviews and remediation planning |
| [deep-dive.md](references/deep-dive.md) | Implementation specifications |
| [test-plan.md](references/test-plan.md) | Translating threats into verification work |
| [service-review.md](references/service-review.md) | Deep service reviews, coverage, cross-service analysis, and HTML reports |
| [report-template.html](assets/report-template.html) | Look and structure of the HTML report — styling, DFD shapes, severity badges, section order |
| [workflow.svg](assets/workflow.svg) | The workflow diagram in this README |

Edit the Markdown files directly. Preserve the `name` and `description` metadata in `SKILL.md` and keep relative links valid. Re-run your installation command after editing to update the installed copy.

## Publishing on GitHub

Push this folder with `SKILL.md`, `references/`, and `assets/` at the repository root. Add a `LICENSE` file with your chosen terms for reuse. The skill itself needs no build step or package dependencies.

## Host documentation

- [Codex skills](https://learn.chatgpt.com/docs/build-skills)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
- [Gemini CLI skills](https://geminicli.com/docs/cli/skills/)
