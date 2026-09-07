# Secure System Design

An Agent Skill for secure backend design and deep service-by-service architecture reviews. It connects requirements, threat modeling, implementation planning, and human-readable HTML reports.

The skill's invocation name is **`system-design-threat-model`**, as defined in [SKILL.md](SKILL.md).

## What it does

- Designs new backend features, services, and APIs.
- Builds data flow diagrams and identifies trust boundaries.
- Applies STRIDE threat analysis and optional LINDDUN privacy analysis.
- Feeds architectural mitigations back into the design.
- Produces function-level specifications and tests linked to threats.
- Reviews existing systems and produces prioritized remediation plans.
- Reviews all services one by one, tracks coverage, and traces cross-service attack paths.
- Produces an offline HTML report with linked diagrams, threats, and remediation details.

Start with what you already have: an idea, requirements, an HLD, a diagram, code, or a threat list. The workflow starts at the relevant stage.

## Workflow

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

A bounded review defaults to one Markdown document. An all-services review produces engineering Markdown plus an offline `report.html` for human review, with service coverage, DFDs, threat details, and a consolidated remediation backlog. Threat Dragon JSON is an optional export that must be checked against the target version; HTML alone is not a Threat Dragon model. The workflow can also work with material from Threat Dragon, Excalidraw, or a wiki.

### Reviewing all services

1. Inventory services, workers, shared infrastructure, and dependencies; record the review scope and revision.
2. Review each service's entry points and flows in depth, including security controls, failure paths, evidence, and verification tests.
3. Track each service as pending, in progress, reviewed, or blocked. Prioritization changes the order, not the requested coverage.
4. Trace cross-service attack paths and reconcile identity, authorization, shared state, queues, and failure propagation.
5. Deliver engineering Markdown and an offline HTML report with service navigation, linked DFDs and threats, and a deduplicated remediation backlog.

The report separates confirmed findings from unverified threats and records coverage gaps. Proposed fixes remain open until verified. Threat Dragon export is available on request and requires validation against the target version.

This repository contains instructions for the agent to generate these artifacts during a review; it does not include a standalone report application.

## Installation

With Node.js and npm installed, run:

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent codex claude-code hermes-agent gemini-cli
```

This installs the skill for Codex, Claude Code, Hermes Agent, and Gemini CLI. Remove agent names you do not use. `--global` installs for your user across projects; the `--yes` flags skip npm and installer confirmation prompts. The syntax and agent names follow the [Skills CLI documentation](https://github.com/vercel-labs/skills).

To install for just one agent, for example Codex:

```sh
npx --yes skills add Thanarak-q/secure-system-design --global --yes --agent codex
```

To inspect the repository's skills without installing:

```sh
npx --yes skills add Thanarak-q/secure-system-design --list
```

### Usage

In Codex:

```text
$system-design-threat-model review my system architecture
```

In Claude Code:

```text
/system-design-threat-model review my system architecture
```

In Hermes Agent or Gemini CLI:

```text
Use system-design-threat-model to review my system architecture.
```

Restart your agent if the skill does not appear; Gemini CLI also supports `/skills reload`. These instructions target the coding tools, not consumer chat websites. Installer support does not imply that the skill's behavior has been tested in every host.

## Example prompts

**Review all services in depth**

```text
Use system-design-threat-model to review all services in this repository.
Inventory them first, then deeply review each service and its flows.
Finish with cross-service attack paths and an offline HTML report.
Track any blocked or unfinished coverage explicitly.
```

**Design a new feature**

```text
Use system-design-threat-model to design a backend API for file uploads.
Start with requirements and identify the information you need from me.
```

**Review an existing service**

```text
Use system-design-threat-model to review this service's authentication
and file-access flows. Produce a prioritized remediation plan and label
unverified assumptions clearly.
```

**Continue from an HLD**

```text
Use system-design-threat-model with this HLD. Build the data flow diagram,
identify threats, and update the architecture with the required mitigations.
```

**Plan implementation**

```text
Use system-design-threat-model to turn this threat model and its mitigations
into implementation specifications and a test plan.
```

## Files and customization

```text
secure-system-design/
├── README.md
├── SKILL.md
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

Edit the Markdown files directly. Preserve the `name` and `description` metadata in `SKILL.md` and keep relative links valid. Re-run your installation command after editing to update the installed copy.

## Publishing on GitHub

Push this folder with `SKILL.md` and `references/` at the repository root. Add a `LICENSE` file with your chosen terms for reuse. The skill itself needs no build step or package dependencies.

## Host documentation

- [Codex skills](https://learn.chatgpt.com/docs/build-skills)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
- [Gemini CLI skills](https://geminicli.com/docs/cli/skills/)
