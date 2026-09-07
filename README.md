# Secure System Design

An Agent Skill for designing backend systems and reviewing existing architecture. It connects requirements, high-level design, threat modeling, implementation specifications, and test planning.

The skill's invocation name is **`system-design-threat-model`**, as defined in [SKILL.md](SKILL.md).

## What it does

- Designs new backend features, services, and APIs.
- Builds data flow diagrams and identifies trust boundaries.
- Applies STRIDE threat analysis and optional LINDDUN privacy analysis.
- Feeds architectural mitigations back into the design.
- Produces function-level specifications and tests linked to threats.
- Reviews existing systems and produces prioritized remediation plans.

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

The default output is one Markdown document. The workflow can also work with material from Threat Dragon, Excalidraw, or a wiki.

## Installation

Download or clone this repository. Run the commands from its root, where `SKILL.md` and `references/` are located. The installed directory uses the skill's declared name.

### Codex

```sh
mkdir -p ~/.agents/skills/system-design-threat-model
cp -R SKILL.md references ~/.agents/skills/system-design-threat-model/
```

Invoke it:

```text
$system-design-threat-model review my system architecture
```

Codex detects skill changes automatically. Restart it if the skill does not appear.

### Claude Code

```sh
mkdir -p ~/.claude/skills/system-design-threat-model
cp -R SKILL.md references ~/.claude/skills/system-design-threat-model/
```

Invoke it:

```text
/system-design-threat-model review my system architecture
```

### Gemini CLI

```sh
mkdir -p ~/.gemini/skills/system-design-threat-model
cp -R SKILL.md references ~/.gemini/skills/system-design-threat-model/
```

Run `/skills reload` in an existing session, then ask:

```text
Use system-design-threat-model to review my system architecture.
```

The copy commands update files in an existing installation. Keep only one installed copy per tool to avoid duplicate discovery. These instructions target the coding tools, not consumer chat websites. The skill uses their supported Markdown folder format; behavior has not been tested across all three hosts.

## Example prompts

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
    └── test-plan.md
```

| File | Edit to change |
| --- | --- |
| [SKILL.md](SKILL.md) | Main workflow, activation description, and output guidance |
| [hld.md](references/hld.md) | Architecture and design decisions |
| [threat-modeling.md](references/threat-modeling.md) | DFDs, threats, and mitigation decisions |
| [existing-system.md](references/existing-system.md) | Existing-system reviews and remediation planning |
| [deep-dive.md](references/deep-dive.md) | Implementation specifications |
| [test-plan.md](references/test-plan.md) | Translating threats into verification work |

Edit the Markdown files directly. Preserve the `name` and `description` metadata in `SKILL.md` and keep relative links valid. Re-run your installation command after editing to update the installed copy.

## Publishing on GitHub

Push this folder with `SKILL.md` and `references/` at the repository root. Add a `LICENSE` file with your chosen terms for reuse. The skill itself needs no build step or package dependencies.

## Host documentation

- [Codex skills](https://learn.chatgpt.com/docs/build-skills)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
- [Gemini CLI skills](https://geminicli.com/docs/cli/skills/)
