# MigMaster

> A Claude Code based software development methodology toolkit, enforcing industrial-grade commit discipline and testing rigor for AI-assisted compiler development.

MigMaster adapts the contribution conventions of LLVM, GCC, Triton, and IREE into a suite of executable artifacts — Skills, Hooks, and Slash Commands — that constrain AI code generation to the standards of hand-crafted engineering. It is designed for researchers and graduate students who use AI tools to explore compiler code but refuse to let AI erode their understanding of the codebase.

## Problem

AI coding agents produce large, monolithic changes. You ask for a feature, you get 500 lines. The code might work, but you cannot audit it, cannot trace the design decisions, and cannot recover context six months later. In compiler development, this problem is lethal: hand-written IR tests pass, end-to-end tests fail, and the gap between "idealized" and "real" IR grows silently.

## Approach

MigMaster treats AI output as **raw material**, not finished product. It provides:

- **Gatekeeping Hooks** that remind you to run tests and review before commit
- **Workflow Commands** that decompose large tasks into independently verifiable, ≤ 150-line atomic commits
- **Testing Skills** that detect "abstraction degradation" — when hand-written IR drifts too far from real-world SYCL/template-expanded IR
- **Learning Scaffolding** that transforms repository commit histories into structured workbooks with executable experiments

The core philosophy: **AI generates, human distills.** The commit history becomes the permanent documentation, each small commit a verifiable, comprehensible unit of knowledge.

## Documentation

| Document | Purpose |
|----------|---------|
| [`docs/Principle/COMMIT_PRINCIPLES.md`](docs/Principle/COMMIT_PRINCIPLES.md) | Commit/PR rules derived from LLVM, GCC, Triton, IREE |
| [`docs/Principle/TESTING_PYRAMID.md`](docs/Principle/TESTING_PYRAMID.md) | Five-layer testing strategy for compiler projects |
| [`docs/Principle/DOC_CLASSIFY.md`](docs/Principle/DOC_CLASSIFY.md) | Documentation classification and maintenance policy |
| [`docs/Discussion/1-Motivation.md`](docs/Discussion/1-Motivation.md) | Problem statement and motivation |
| [`docs/FormalDesign-v1.md`](docs/FormalDesign-v1.md) | Complete artifact specification (15 artifacts) |

## Artifacts at a Glance

| Layer | Artifacts |
|-------|-----------|
| **Constitution** | `CLAUDE.md` — engineering norms, environment config, workflow routing |
| **Defense Line** | `Stop` Hook — post-task test reminder |
| **Development Flow** | `/pre-plan`, `/micro`, `/commit-review`, `/verify` |
| **Testing & Analysis** | `/test-gen` (4-quadrant generation + abstraction degradation detection + IR reduction), `/split-commit` |
| **Knowledge** | `/mentor`, `/learn-from` (workbook generator), `/archeology` (repo archaeology), `/adr` |
| **Reflection** | `/check-turn` (pivot signal detection), `/config` (descriptive CLAUDE.md editing) |

See [`docs/FormalDesign-v1.md`](docs/FormalDesign-v1.md) for full functional boundaries, input/output contracts, and edge cases for each artifact.

## Quick Start

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- LLVM 19+ (for IR reduction and testing workflows)
- Python 3.8+

### Installation

```bash
# Clone into any existing project you want to apply the methodology to
git clone https://github.com/your-org/MigMaster.git
cd MigMaster

# The .claude/ directory contains skills and hooks.
# CLAUDE.md will be loaded automatically by Claude Code in this directory.
```

### First Steps

1. **Read the motivation**: [`docs/Discussion/1-Motivation.md`](docs/Discussion/1-Motivation.md)
2. **Configure the environment**: Run `/config` in Claude Code to set your LLVM build path and toolchain locations
3. **Start with a small task**: Use `/pre-plan` to validate task granularity before coding
4. **Execute atomically**: Use `/micro` for each ≤150-line implementation step
5. **Review before commit**: Use `/commit-review` for pre-commit inspection
6. **Verify after commit**: Use `/verify` to generate manually executable test commands

## Project Structure

```
MigMaster/
├── CLAUDE.md                         # Project constitution (to be implemented)
├── README.md                         # This file
├── .claude/
│   └── settings.local.json           # Hook configurations
│   └── skills/                       # Skill definitions (to be implemented)
│       ├── pre-plan/SKILL.md
│       ├── micro/SKILL.md
│       ├── commit-review/SKILL.md
│       ├── verify/SKILL.md
│       ├── test-gen/SKILL.md
│       ├── split-commit/SKILL.md
│       ├── adr/SKILL.md
│       ├── archeology/SKILL.md
│       ├── mentor/SKILL.md
│       ├── learn-from/SKILL.md
│       ├── check-turn/SKILL.md
│       └── config/SKILL.md
└── docs/
    ├── FormalDesign-v1.md            # Artifact specification
    ├── Principle/                    # Engineering principles
    │   ├── COMMIT_PRINCIPLES.md
    │   ├── TESTING_PYRAMID.md
    │   └── DOC_CLASSIFY.md
    └── Discussion/                   # Design discussion archive
        ├── 1-Motivation.md
        ├── 2-ARTIFACT_ANALYSIS.md
        ├── 3-Reply.md
        ├── 4-Analysis.md
        └── 5-FinalDesign.md
```

## Implementation Status

- [x] Engineering principles documented
- [x] Artifact formal specification (v1)
- [ ] CLAUDE.md (P0)
- [ ] Hooks (P1)
- [ ] Core commands: `/pre-plan`, `/micro`, `/commit-review`, `/verify` (P1)
- [ ] Analysis skills: `/test-gen`, `/split-commit` (P2)
- [ ] Knowledge skills: `/learn-from`, `/adr`, `/config` (P2)
- [ ] Extended skills: `/archeology`, `/mentor`, `/check-turn` (P3)

## Design Principles

1. **AI generates, human distills.** Every line entering the handcrafted library passes through human judgment.
2. **One commit, one purpose.** No mixed bugfixes, features, and refactors in the same commit.
3. **Tests are non-negotiable.** Feature tests define contracts; regression tests defend against real-world complexity.
4. **Revert-to-green.** A broken build is always reverted first, investigated second.
5. **Small and verifiable.** PRs ≤ 200 lines; each commit independently compiles and passes tests.

## License

Apache 2.0 — aligned with the LLVM ecosystem this toolkit draws from.

## Acknowledgments

The engineering practices codified here are derived from the contribution guidelines of:

- [LLVM](https://llvm.org/docs/DeveloperPolicy.html)
- [GCC](https://gcc.gnu.org/contribute.html)
- [Triton](https://github.com/triton-lang/triton)
- [IREE](https://iree.dev/developers/general/contributing/)
