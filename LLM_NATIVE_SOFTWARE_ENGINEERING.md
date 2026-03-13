# LLM-Native Software Engineering

## A Protocol for Coordinating Human and Artificial Intelligence in Software Development

**Version:** 4.0
**Author:** Pragnakar Pedapenki
**Repository:** https://github.com/pragnakar/LLM_NATIVE_SOFTWARE_ENGINEERING
**Applicability:** Claude Code, Cursor, Windsurf, GitHub Copilot, Aider, and any LLM-based coding system
**Companion Meta-Prompts:** Deployment Engineering, DevOps, Database, UI-UX, Security Engineering, MLOps, API Design, Testing Strategy, Documentation, Scrum

---

## Thesis

LLM-native software engineering requires explicit coordination protocols between agents — human or artificial.

These protocols take the form of three control documents: a **directive** (culture), a **specification** (contract), and a **build log** (memory). Together, they function as a communication substrate that enables reliable software construction regardless of which agents — human developers, AI coding assistants, or autonomous agent swarms — are doing the building.

Software development with LLMs is becoming a **protocol problem**, not a coding problem.

---

## What This Document Is

This document serves as a **meta-prompt** — a prompt that runs before the actual task prompt. Its purpose is not to perform a task, but to initialize the environment in which the task will run.

Instead of:

```
task prompt → AI response
```

The workflow becomes:

```
meta-prompt (environment and protocol initialization)
     ↓
task prompt (what to build)
     ↓
AI execution within that structured environment
```

The meta-prompt defines workflow structure, behavioral constraints, verification discipline, state tracking, and architectural rules. It establishes the protocol for how AI and humans collaborate.

This methodology was developed through the construction of a non-trivial software project (SAGE — a 7-phase, multi-package MCP server) built from blank repository to shipped v0.1.0 using a two-instance architecture where a conversational AI designed the system and a coding AI implemented it. Every element in this document earned its place by catching a defect, preventing a regression, or saving significant rework during that build.

---

## Part I — The Methodology

---

### 1. The Core Insight

LLM coding agents are strong implementers but unreliable architects.

| Role | LLM Performance |
|---|---|
| Architect / system thinker | Weak — inconsistent |
| Implementer | Strong |
| Refactorer | Strong |
| Tester (unit) | Moderate — biased toward happy paths |
| Integration verifier | Weak without explicit prompting |

This asymmetry produces a characteristic failure mode: **local correctness with global incoherence.** The agent writes `db.py`, `cache.py`, and `api.py` — each individually correct, but collectively violating the architecture.

The solution is separation of concerns:

```
┌──────────────────────────────────────┐
│  ARCHITECT LAYER                      │
│  (Conversational AI or Human)         │
│                                       │
│  • Designs architecture               │
│  • Writes specifications              │
│  • Creates phase prompts              │
│  • Reviews and approves               │
│  • Makes trade-off decisions          │
│  • Resolves ambiguity                 │
└────────────────┬─────────────────────┘
                 │  Control Documents
                 ▼
┌──────────────────────────────────────┐
│  BUILDER LAYER                        │
│  (LLM Coding Agent)                  │
│                                       │
│  • Reads specifications               │
│  • Implements bounded tasks           │
│  • Writes tests                       │
│  • Updates build log                  │
│  • Reports status                     │
│  • Never makes architectural decisions│
└──────────────────────────────────────┘
```

The Architect produces control documents. The Builder executes them. The Builder never decides *what* to build — only *how* to implement what was specified.

---

### 2. The Three Control Documents

Every project requires three control documents. They serve as the communication protocol between architect and builder — analogous to APIs between services or network protocols between machines.

| Document | Distributed System Analogy | Function |
|---|---|---|
| AGENT.md (Directive) | Runtime policy | Culture — how the agent behaves |
| SPEC.md (Specification) | Interface contract | Contract — what to build |
| BUILD_LOG.md (Build Log) | System state | Memory — where we are |

#### 2.1 The Directive — AGENT.md

**Purpose:** Tells the coding agent WHO it is, WHAT it's building, and HOW to behave.

**Platform-specific naming:**

| Platform | Filename |
|---|---|
| Claude Code | `CLAUDE.md` |
| Cursor | `.cursorrules` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Universal | `AGENT.md` |

**Contents:**

```markdown
# [PROJECT] — Agent Instructions

## Project
One-paragraph description.

## Read First
"Before writing any code, read SPEC.md."

## Architecture
High-level structure. Module responsibilities.

## Tech Stack
Languages, frameworks, libraries, exact versions.

## Build Order
Numbered phase sequence.

## Critical Design Rules
Non-negotiable constraints.

## Code Style
Type hints, docstrings, naming, logging.

## Testing
How to run tests. Known correct values.

## Dependencies
Package names and minimum versions.

## What NOT To Do
Explicit prohibitions.
```

**Critical constraint: Keep under 150 lines.** The agent reads this at the start of every session. If it's too long, important instructions get lost. Put details in the SPEC.

**Why it matters:** Without this file, every new session makes its own decisions about structure, naming, and error handling — inconsistently. The directive is the project's culture.

#### 2.2 The Specification — SPEC.md

**Purpose:** Complete technical specification. The architect's design made explicit enough for an LLM to implement without clarifying questions.

**For small-to-medium projects:** A single `SPEC.md` works.

**For larger projects:** Use modular specifications to prevent spec drift:

```
.build/spec/
├── architecture.md
├── data_models.md
├── api.md
├── modules/
│   ├── solver.md
│   ├── parser.md
│   └── server.md
└── schemas/                    ← Machine-readable contracts
    ├── solver_input.json       ← JSON Schema
    ├── api.yaml                ← OpenAPI
    └── config.schema.json
```

Machine-readable schemas (JSON Schema, OpenAPI, protobuf) provide a layer of truth that can be validated automatically. Parts of the specification should be **executable, not just readable** — this prevents spec drift, where the spec diverges from the codebase after several phases.

**Spec versioning:** Track changes as the spec evolves during build:

```markdown
## Spec Changelog
- v1.1 (Phase 3): Added objective_quadratic to SolverInput
- v1.2 (Phase 5): Changed relaxation ranking to percentage-based
```

#### 2.3 The Build Log — BUILD_LOG.md

**Purpose:** Persistent memory across sessions. The coding agent's state file.

**The critical instruction that makes it work:**

> "If BUILD_LOG.md already exists, READ IT FIRST before doing anything. Resume from where the log says you left off. Do not repeat completed work."

This single instruction enables session recovery across credit limits, timeouts, and context window resets.

**Preventing bloat:** When the log exceeds ~300 lines, compress completed phases into summaries and archive details:

```
.build/state/
├── BUILD_STATE.md        ← Current status (< 50 lines, always read)
├── BUILD_SUMMARY.md      ← Compressed phase history (< 200 lines)
└── archive/
    ├── phase_1_log.md    ← Full detail for completed phases
    └── phase_2_log.md
```

The agent reads `BUILD_STATE.md` (tiny, always current). Full logs exist for human review but aren't loaded into agent context.

---

### 3. The Phase Protocol

Software is built in sequential phases. Each phase follows an identical four-step protocol:

```
PHASE PROMPT ──▶ BUILD ──▶ VERIFY ──▶ APPROVE ──▶ next phase
(architect)      (agent)   (agent)    (human)
```

#### 3.1 Phase Prompt

Pre-written by the architect. Specifies:
- Git branch name
- Exact scope (files, functions)
- Spec section references (not repeating content)
- Test requirements with known-correct values
- Commit conventions
- Stop condition: "Do not proceed until I review and approve"

#### 3.2 Build

The agent creates the branch, implements code and tests, runs tests, commits, updates the build log, and reports completion.

#### 3.3 Verification Prompt

**The methodology's most valuable innovation.** A separate prompt, run after the build, that tests what build-phase tests cannot:

| Build Tests | Verification Tests |
|---|---|
| Does this module work? | Does it work WITH previous modules? |
| Unit correctness | Cross-phase integration |
| Happy path | Error paths and edge cases |
| Schema validity | JSON serialization roundtrips |
| Current phase only | Full pipeline Phase 1 through N |
| Pass/fail | Quality (readable? professional?) |

**Evidence — bugs caught exclusively by verification, invisible to unit tests:**

| Phase | Bug | Impact if Missed |
|---|---|---|
| 2 | Binary variable bounds silently ignored | Wrong scheduling assignments |
| 2 | `float('inf')` breaks JSON serialization | Silent data loss in transport |
| 4 | `ffill()` bleeds description rows into data | Wrong optimization inputs |
| 5 | Consecutive days RHS wrong for multi-shift | Feasible models appear infeasible |
| 6 | Console entry point references wrong module | Install works, command fails |
| 7 | Example columns don't match parser | Every example fails on first try |
| 7 | Percentages instead of fractions in examples | Examples always return infeasible |
| 7 | pyproject.toml key ordering breaks pip | Installation fails entirely |

Unit tests are not sufficient validators for AI-generated systems. Cross-boundary verification is the strongest argument in this entire methodology.

#### 3.4 Approval Gate

The human reviews the verification report and approves, requests fixes, or revises the plan. This gate prevents compounding errors across phases.

---

### 4. Git Strategy

```
main ← merges from develop at milestones only
  └── develop ← integration branch
        ├── feature/phase-1-[name]    → merge after verification
        ├── feature/phase-2-[name]
        └── ...

Tags: phase-N-complete (rollback points), vX.Y.Z (releases)
```

Phase tags enable rollback. If Phase 5 corrupts Phase 3's work, you reset to `phase-3-complete` and rebuild from there.

---

### 5. Naming Conventions — Phases, Stages, Versions

A project has three independent timelines that evolve at different rates. Conflating them produces confusion in build logs, git history, and team communication. Each timeline has its own naming scheme, its own cadence, and its own success criteria.

```
SPECIFICATIONS          BUILDS                  RELEASES
(vision & design)       (construction)          (shipped artifacts)

evolve in PHASES        progress through        tracked by VERSIONS
                        STAGES

Phase 1: Data Models    Stage: implement        v0.1.0-alpha
Phase 2: Core Engine    Stage: test             v0.1.0-beta
Phase 3: API Layer      Stage: verify           v0.1.0
Phase 4: Integration    Stage: approve          v0.2.0
                        Stage: failed → rework  v1.0.0
```

#### 5.1 Specifications Evolve in Phases

A **phase** is a unit of architectural progress. It represents a design decision that has been specified, scoped, and approved for implementation. Phases are sequential and cumulative — Phase 3 builds on the foundation established by Phases 1 and 2.

**Naming:** `phase-N-[descriptive-name]`

**Examples:**
- `phase-1-data-models` — schema and type definitions
- `phase-2-solver-engine` — core computation module
- `phase-3-api-layer` — REST endpoints and serialization
- `phase-4-integration` — cross-module wiring and pipeline

**Key property:** A phase is a *design* concept. It describes what the specification calls for, not whether the code compiles. A phase can be fully specified before any code is written. Phase numbering never resets — it is the permanent record of the project's architectural evolution.

**Spec versioning within phases:** As specifications evolve, they carry their own version:

```
SPEC.md v1.0 — Initial specification (Phases 1-4 defined)
SPEC.md v1.1 — Phase 3 revised: added pagination to API
SPEC.md v1.2 — Phase 5 added: caching layer
SPEC.md v2.0 — Major redesign: event-driven architecture
```

#### 5.2 Builds Progress Through Stages

A **stage** is a unit of construction work within a phase. It represents an attempt to implement, test, or verify the phase's specification. Stages are where work happens — and where work fails. Not every stage succeeds. Failed stages are recorded, diagnosed, and retried.

**Naming:** `stage:[action]` within a phase context

**The standard stage sequence for each phase:**

```
phase-N
  ├── stage: implement     ← Agent writes code and unit tests
  ├── stage: test          ← Unit tests executed, linting passed
  ├── stage: verify        ← Cross-phase integration verification
  ├── stage: approve       ← Human reviews and approves
  └── stage: merge         ← Branch merged, phase tag applied
```

**Failed stages are first-class events:**

```
phase-3-api-layer
  ├── stage: implement     ✓ completed
  ├── stage: test          ✓ passed (23/23 unit tests)
  ├── stage: verify        ✗ FAILED — JSON serialization breaks on inf values
  ├── stage: rework        ✓ completed (fixed float handling)
  ├── stage: verify        ✓ passed (re-verified phases 1-3)
  ├── stage: approve       ✓ human approved
  └── stage: merge         ✓ merged to develop
```

**Key property:** Stages are *execution* concepts. They describe what happened during construction — including failures. A phase might pass through its stages in one clean run, or it might cycle through implement → test → fail → rework → test → verify multiple times. The build log records every stage transition, including failures, because failure history is diagnostic data.

**Build log entries reference both phase and stage:**

```markdown
## Phase 3 — API Layer
### Stage: implement (2026-03-13 09:00)
- Created api/routes.py, api/serializers.py
- 23 unit tests written

### Stage: test (2026-03-13 10:15)
- All 23 tests passing
- ruff: no violations

### Stage: verify (2026-03-13 10:45)
- FAILED: float('inf') in solver output breaks json.dumps()
- Root cause: Phase 2 solver returns inf for unbounded variables

### Stage: rework (2026-03-13 11:00)
- Added inf/nan sanitizer in serialization layer
- 2 new tests for edge cases

### Stage: verify (2026-03-13 11:30)
- PASSED: Full pipeline Phase 1→3 verified
- JSON roundtrip: all types serialize and deserialize correctly

### Stage: approve (2026-03-13 12:00)
- Human reviewed verification report
- Approved for merge

### Stage: merge (2026-03-13 12:05)
- Merged feature/phase-3-api-layer → develop
- Tag: phase-3-complete
```

#### 5.3 Releases Are Tracked by Versions

A **version** is a shipped artifact — a point-in-time snapshot of the codebase that is tagged, packaged, and made available to users. Versions follow [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`.

**Naming:** `vMAJOR.MINOR.PATCH[-prerelease]`

**Examples:**
- `v0.1.0-alpha` — first functional prototype, internal testing only
- `v0.1.0-beta` — feature-complete for initial scope, external testing
- `v0.1.0` — first stable release
- `v0.2.0` — new features added (minor)
- `v1.0.0` — production-ready, API contract stable (major)

**Version boundaries:**

| Version Component | Changes When | Example |
|---|---|---|
| MAJOR (1.x.x) | Breaking changes to public API or behavior | v1.0.0 → v2.0.0 |
| MINOR (x.1.x) | New features, backward compatible | v1.0.0 → v1.1.0 |
| PATCH (x.x.1) | Bug fixes, no new features | v1.0.0 → v1.0.1 |
| Pre-release | Testing stages before stable | v0.1.0-alpha, v0.1.0-beta |

**Key property:** Versions are *delivery* concepts. They describe what users receive. A version may span multiple phases (v0.1.0 might ship after Phase 4 completes) or a single phase might produce multiple versions (Phase 5 hotfixes produce v0.1.1 and v0.1.2). Versions and phases are deliberately decoupled.

#### 5.4 The Three Timelines Are Independent

This independence is the critical design decision. Each vertical evolves on its own schedule:

```
Spec Timeline:    Phase 1 ──── Phase 2 ──── Phase 3 ──── Phase 4 ────
                       │            │            │            │
Build Timeline:   [impl→test→  [impl→test→  [impl→test→  [impl→test→
                   verify→      fail→rework→  verify→      verify→
                   approve→     verify→       approve→     approve→
                   merge]       approve→      merge]       merge]
                                merge]
                       │                         │            │
Release Timeline:      ·            ·        v0.1.0-alpha    ·
                       ·            ·            ·        v0.1.0
```

**Why this matters:**

- **A spec phase does not require a release.** Phase 2 might be pure infrastructure with no user-visible change. It has phases and stages but no version.
- **A release does not require a new phase.** A hotfix producing v0.1.1 happens within the existing phase structure — it is a patch, not new architectural work.
- **A failed build stage does not reset the phase.** Phase 3's verification failure and rework are stages within Phase 3, not a new Phase 3.1. The phase is the design scope; stages track the execution within that scope.
- **Spec versions, phase numbers, and release versions are different numbers.** SPEC.md v1.2 is not related to Phase 2 or release v1.2.0. Using the same number for different concepts guarantees confusion.

#### 5.5 Naming Convention Reference

| Concept | Naming Pattern | Example | Tracked In |
|---|---|---|---|
| Spec version | `vN.N` | SPEC.md v1.2 | Spec Changelog |
| Phase | `phase-N-[name]` | phase-3-api-layer | BUILD_LOG.md, git branch |
| Stage | `stage: [action]` | stage: verify ✗ FAILED | BUILD_LOG.md |
| Git branch | `feature/phase-N-[name]` | feature/phase-3-api-layer | Git |
| Phase tag | `phase-N-complete` | phase-3-complete | Git tag |
| Release version | `vMAJOR.MINOR.PATCH` | v0.1.0 | Git tag, package registry |
| Pre-release | `vM.N.P-[label]` | v0.1.0-alpha | Git tag |

---

### 6. Pre-Commit Security

Before pushing code to any remote repository, verify that commits do not contain sensitive information:

- API keys or access tokens
- Authentication credentials (passwords, private keys, secrets)
- Environment configuration files containing secrets
- Personally identifiable information (PII)
- Internal system identifiers or confidential infrastructure details

Automated secret-scanning tools should be used whenever possible. The only exceptions are explicit contact information intentionally included for support or documentation, such as project email addresses and maintainers' public contact information.

This check should be part of the pre-commit hook configuration established in Phase 1 of any project build.

---

## Part II — Known Limitations

---

### 7. Where This Methodology Breaks

#### 7.1 Spec Drift

After 3-4 phases, monolithic specs diverge from the codebase. **Mitigation:** Modular specs, machine-readable schemas validated in CI, spec changelog.

#### 7.2 Waterfall Rigidity

Sequential phases don't accommodate mid-build redesign. **Mitigation:** An iterative loop — spec → build → verify → revise spec → continue. Phase prompts should anticipate revision.

#### 7.3 Architect Bottleneck

One human architect writing all specs becomes a bottleneck. **Mitigation:** Module-level spec fragments. Eventually, a Spec Generator agent produces specifications from high-level intent, reviewed by the human architect before builders execute.

#### 7.4 LLM-Dependent Verification

Builder and verifier share the same model's blind spots. **Mitigation:** Layered verification:

| Layer | Tool |
|---|---|
| Static analysis | mypy, ruff, eslint |
| Schema validation | JSON Schema, OpenAPI validators |
| Unit tests | pytest, jest |
| Integration tests | Verification prompts |
| Deterministic CI | GitHub Actions pipeline |
| Review agent | Separate model instance or temperature |

Verification prompts should progressively become CI jobs.

#### 7.5 Build Log Bloat

Large logs exceed context windows. **Mitigation:** Split into BUILD_STATE.md (current, tiny), BUILD_SUMMARY.md (compressed), and archived phase logs.

---

## Part III — The Future Architecture

---

### 8. Evolution of the .build System

#### 8.1 Current: Static Documents

```
.build/
├── AGENT.md
├── SPEC.md
├── BUILD_LOG.md
├── PHASE_PROMPTS.md
└── VERIFICATION_PROMPTS.md
```

Sufficient for single-developer, single-project builds.

#### 8.2 Near-Term: Active Tooling

```
.build/
├── docs/                           ← Specifications and decisions
│   ├── AGENT.md
│   ├── spec/                       ← Modular specs
│   │   ├── architecture.md
│   │   └── modules/
│   ├── schemas/                    ← Machine-readable contracts
│   │   ├── api.yaml
│   │   └── data_models.json
│   └── decisions/                  ← Architectural Decision Records
│       ├── ADR-001-solver-choice.md
│       └── ADR-002-api-design.md
│
├── state/                          ← Build state (summarized)
│   ├── BUILD_STATE.md
│   ├── BUILD_SUMMARY.md
│   └── archive/
│
├── prompts/                        ← Phase and verification prompts
│   ├── phases/
│   ├── verification/
│   └── templates/                  ← Reusable prompt templates
│
├── scripts/                        ← Automation
│   ├── verify_phase.py
│   ├── summarize_log.py
│   └── validate_schemas.py
│
├── eval/                           ← Quality evaluation
│   ├── benchmark_problems/
│   └── quality_rubrics.md
│
└── telemetry/                      ← Agent performance tracking
    ├── token_usage.csv
    └── failure_modes.md
```

**Architectural Decision Records (ADRs)** provide historical reasoning that code cannot express. When a new agent session encounters code and wonders "why was it built this way?", ADRs answer.

**Agent telemetry** tracks token usage, failure modes, and retry patterns. This transforms .build from a control system into a learning system.

#### 8.3 Medium-Term: Externalized State

As projects scale, the .build directory becomes a lightweight interface to a cloud-hosted build state:

```
.build/
├── config.yaml            ← Connection to remote build state
├── project_id             ← Workspace identifier  
├── .cache/                ← Cached summaries for offline work
├── schemas/               ← Still local (source of truth)
└── spec/                  ← Still local (source of truth)
```

All persistent workflow data — build logs, phase history, verification results, specification versions, agent execution traces — resides in a centralized cloud database. The repository remains focused on source code.

This architecture prevents repository bloat, enables real-time collaboration between developers and agents, supports scalable querying of build history, and allows centralized governance for large organizations.

#### 8.4 Integration with Agile Planning Systems

The .build system exposes integration layers to common planning tools:

| System | Integration |
|---|---|
| Jira / Linear / ClickUp | Phase prompts ↔ sprint tickets (bi-directional) |
| Notion | Spec sections ↔ documentation pages |
| GitHub Issues | Verification failures → auto-created issues |
| Slack / Teams | Phase completion → notifications |
| CI/CD | Verification prompts → automated pipeline stages |

This connects agent-driven development with traditional agile planning, ensuring AI-assisted work remains visible within standard organizational processes.

---

### 9. Agent Development Runtime (ADR)

As the .build system matures, the next logical evolution is transforming it from a static directory into a **runtime environment** that manages and orchestrates software-building agents — an **Agent Development Runtime**.

The ADR acts as the operating system for AI-assisted development workflows. Instead of manually prompting agents and coordinating through documents, the runtime manages agents, tracks system state, executes workflows, and enforces architectural constraints.

#### 9.1 Why a Runtime Is Necessary

| Problem | Impact |
|---|---|
| Stateless sessions | Agents forget progress between runs |
| Manual orchestration | Humans coordinate agent tasks by pasting prompts |
| Fragmented state | Build history scattered across sessions |
| Inconsistent verification | Quality checks vary across sessions |
| Poor multi-agent coordination | Parallel work becomes chaotic |

The .build methodology addresses these with structured documents. A runtime system allows these protocols to be **executed automatically** rather than enforced manually.

#### 9.2 Core Capabilities

**Agent Orchestration:**

| Agent Type | Responsibility |
|---|---|
| Spec Generator Agent | Produces specifications from high-level intent |
| Architect Agent | Updates system specifications and resolves conflicts |
| Builder Agent | Implements modules according to spec |
| Review Agent | Detects architectural drift and spec violations |
| Verification Agent | Runs integration tests and validations |
| Refactor Agent | Improves code quality and structure |
| Documentation Agent | Maintains developer documentation |

Each agent operates within constraints derived from the project's specification and directive files.

**Persistent Development State:**

A centralized project state store (cloud-hosted NoSQL database or knowledge graph) containing build logs, phase progress, specification versions, architectural decisions, verification results, agent execution traces, and cost metrics. The local .build directory becomes a thin interface to this state.

**Workflow Execution Engine:**

Instead of manually pasting prompts, the runtime executes development workflows as structured pipelines:

```
SPEC CREATED
     ↓
PHASE GENERATED (by Spec Generator Agent)
     ↓
BUILDER AGENTS EXECUTE
     ↓
VERIFICATION PIPELINE
     ↓
REVIEW AGENTS
     ↓
HUMAN APPROVAL (or Orchestrator approval for routine phases)
     ↓
NEXT PHASE
```

**Build State Snapshots:**

Git-like commits for the entire development workflow, not just code. A snapshot captures specification version, build progress, architectural decisions, agent configurations, and verification results. This enables reproducible builds, workflow rollback, experimentation with different agent strategies, and comparison of development approaches.

#### 9.3 Security Model for Agentic Development

Once agents run builds autonomously, new security concerns emerge:

| Threat | Description |
|---|---|
| Prompt injection | Malicious input altering agent behavior |
| Malicious code generation | Agent producing harmful code |
| Spec tampering | Unauthorized modification of specifications |
| Credential leaks | Secrets exposed in generated code or logs |
| Supply chain attacks | Compromised dependencies introduced by agents |

The ADR must implement:

- **Agent identity and authentication:** Each agent has verified credentials
- **Permission scopes:** Builders can modify code but not specs; architects can modify specs but not deploy
- **Prompt sanitization:** Input validation before agent execution
- **Build provenance:** Cryptographic chain of custody for code changes
- **Secret scanning:** Automated detection of credentials in generated code
- **Audit logging:** Complete record of agent actions for security review

**Permission model:**

| Role | Permissions |
|---|---|
| Architect | Modify specifications, approve phases |
| Builder | Implement code, update build state |
| Verifier | Execute tests, submit verification reports |
| Reviewer | Evaluate architectural compliance |
| Observer | Read-only access to project state |

#### 9.4 System Architecture

```
┌──────────────────────────────────────────┐
│          HUMAN INTERFACE                  │
│   CLI / IDE Plugin / Web Dashboard       │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│       AGENT DEVELOPMENT RUNTIME           │
│                                           │
│  • Agent Orchestrator                     │
│  • Workflow Engine                        │
│  • Spec Manager & Version Control         │
│  • Verification Pipeline                  │
│  • Security & Access Control              │
│  • Telemetry & Cost Metrics               │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│        PROJECT STATE DATABASE             │
│        (NoSQL / Knowledge Graph)          │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│           AGENT WORKERS                   │
│                                           │
│  Spec Generator │ Builder │ Reviewer      │
│  Verifier │ Refactor │ Documentation      │
└──────────────────────────────────────────┘
```

The ADR represents the next layer in the software tooling stack:

```
Traditional:         Future:

IDE                  IDE
Version Control      Agent Development Runtime
CI/CD                Version Control
Cloud                CI/CD
                     Cloud
```

Just as container orchestration transformed infrastructure management, the ADR could become the standard way teams coordinate AI-driven software development.

---

### 10. Cross-Repository Knowledge

Organizations working on multiple projects benefit from a shared architectural knowledge base:

```
org-build-knowledge/
├── patterns/                      ← Reusable architectural patterns
│   ├── mcp-server-pattern.md
│   └── api-service-pattern.md
├── lessons/                       ← Hard-won implementation lessons
│   ├── solver-integration.md
│   └── sdk-gotchas.md
├── templates/                     ← Project scaffolds
│   ├── python-monorepo/
│   └── typescript-api/
└── metrics/
    └── agent-performance-baselines.csv
```

Agents working across projects access common knowledge, reducing duplication and enabling consistent engineering practices. This transforms .build from a per-project tool into an organizational memory system.

---

## Part IV — The Meta-Prompt Ecosystem

---

### 11. The Meta-Prompt Ecosystem

The meta-prompt concept — a structured protocol that initializes an environment before task execution — is not limited to software construction. Software engineering is one domain; many others benefit from the same pattern.

A mature development lifecycle involves multiple domains of expertise, each governed by its own meta-prompt. These meta-prompts operate as complementary layers:

```
┌──────────────────────────────────────────────────────────────┐
│  LLM-Native Software Engineering                              │
│  (Architecture, development methodology,                      │
│   verification discipline, build protocols)                   │
│                                                                │
│  Invokes companion meta-prompts as needed:                    │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬───┘
   │      │      │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
┌──────┐┌──────┐┌──────┐┌─────┐┌──────┐┌─────┐┌─────┐┌─────┐┌─────┐
│Deploy││DevOps││  DB  ││UI/UX││ Sec  ││MLOps││ API ││ Test││ Doc │
│ Eng  ││      ││      ││     ││ Eng  ││     ││Design││Strat││     │
└──────┘└──────┘└──────┘└─────┘└──────┘└─────┘└─────┘└─────┘└─────┘
```

Each meta-prompt is maintained as an independent, evolving repository. They are invoked by the LLM-Native Software Engineering meta-prompt when their domain becomes relevant during a project's lifecycle. Every companion follows the same canonical structure: Thesis → When Invoked → Scope → Principles → Practices → Agent Instructions → Integration Points → Anti-Patterns → Quick-Start Checklist.

#### 11.1 The Complete Ecosystem

| Meta-Prompt | Repository | Responsibility | When Invoked |
|---|---|---|---|
| **LLM-Native Software Engineering** | [GitHub](https://github.com/pragnakar/LLM_NATIVE_SOFTWARE_ENGINEERING) | Architecture, phased development, verification protocols, build-state tracking | Always — this is the parent methodology |
| **Deployment Engineering** | [GitHub](https://github.com/pragnakar/Deployment_Engineering) | Docker environments, Kubernetes, CI/CD pipelines, infrastructure as code, secrets management | Before development begins, when infrastructure must be defined |
| **DevOps** | [GitHub](https://github.com/pragnakar/DevOps) | Runtime operations, monitoring, alerting, incident response, SLOs, reliability engineering | When systems approach production readiness |
| **Database** | [GitHub](https://github.com/pragnakar/Database) | Technology selection, schema design, indexing, query optimization, backup and recovery | When data persistence decisions are required |
| **UI/UX** | [GitHub](https://github.com/pragnakar/UI-UX) | User-centered design, information architecture, accessibility, responsive design, design systems | When user-facing interfaces are being designed or built |
| **Security Engineering** | [GitHub](https://github.com/pragnakar/Security_Engineering) | Threat modeling, secure coding, input validation, cryptography, dependency security, secrets lifecycle | When security requirements need to inform design (always) |
| **MLOps** | [GitHub](https://github.com/pragnakar/MLOps) | Experiment tracking, data versioning, training pipelines, model registry, drift monitoring, ML governance | When the project includes machine learning models |
| **API Design** | [GitHub](https://github.com/pragnakar/API_Design) | Resource modeling, versioning, error handling, pagination, rate limiting, OpenAPI specifications | When APIs are being designed or documented |
| **Testing Strategy** | [GitHub](https://github.com/pragnakar/Testing-Strategy) | Test architecture, coverage strategy, performance testing, contract testing, AI-specific verification | When test planning and coverage standards need to be established |
| **Documentation** | [GitHub](https://github.com/pragnakar/Documentation) | Technical writing, ADRs, API docs, architecture docs, runbooks, documentation-as-code | When documentation architecture needs to be defined |
| **Scrum** | [GitHub](https://github.com/pragnakar/Scrum) | Sprint cycle management, backlog refinement, ceremonies, velocity tracking, Definition of Done | When iterative delivery and sprint planning structure is needed |

#### 11.2 Invocation Sequence

The meta-prompts have a natural invocation order driven by project lifecycle, though any can be invoked at any time:

```
Project Start
     │
     ├── 1. Deployment Engineering    ← Environment before code
     ├── 2. LLM-Native Software Eng   ← Architecture and build protocol
     ├── 3. Database                   ← Data layer design
     ├── 4. API Design                 ← Interface contracts
     ├── 5. Security Engineering       ← Threat model and security controls
     ├── 6. UI/UX                      ← User interface design
     ├── 7. Testing Strategy           ← Test architecture and coverage
     ├── 8. Documentation              ← Docs architecture and ADRs
     ├── 9. Scrum                      ← Sprint cycles and iterative delivery
     │
     ├── During development:
     │   └── MLOps                     ← If ML models are involved
     │
     └── Approaching production:
         └── DevOps                    ← Monitoring, alerting, reliability
```

#### 11.3 Cross-Cutting Relationships

Meta-prompts are not isolated silos — they form a web of explicit integration points. Each companion defines how it connects to every relevant sibling:

```
                    Deployment Engineering
                   ╱          │           ╲
            Security     LLM-Native SE     DevOps
            Engineering   │    │    │      ╱    │
               │    ╲     │    │    │     ╱     │
               │     API Design  Database      │
               │         │    │    │           │
               │    Testing Strategy           │
               │              │                │
               └──── Documentation ────────────┘
```

**Key integration patterns:**

- **Security Engineering** touches every meta-prompt — it defines security controls that all other domains implement
- **Deployment Engineering** and **DevOps** are paired: one provisions infrastructure, the other operates it
- **API Design** and **Database** are paired: API shapes and database schemas inform each other
- **Testing Strategy** validates work produced under every other meta-prompt
- **Documentation** records decisions and knowledge from every domain

#### 11.4 Independent Evolution

Each meta-prompt is versioned and maintained independently. This is a deliberate design choice:

- A new version of **Database** (adding graph database guidance) does not require a new version of **UI/UX**
- A major revision of **Security Engineering** does not force an update to **MLOps**
- Each domain's expertise deepens at its own pace without creating monolithic documents that exceed agent context limits

The parent meta-prompt (this document) defines the methodology and coordination protocol. Companion meta-prompts define domain-specific practices. The pattern is extensible — each new domain adds a new meta-prompt repository that the engineering meta-prompt can invoke.

---

## Part V — The Paradigm Shift

---

### 12. Three Emerging Paradigms

The industry is exploring three distinct approaches. They haven't converged.

#### 12.1 Spec-Driven (This Methodology)

```
Human writes spec → Agent implements → Human verifies → Iterate
```

**Best for:** Greenfield projects, production-grade output, well-understood domains.

#### 12.2 Repo-Evolution (Cursor / Windsurf Style)

```
Human has codebase → Agent modifies in-place → Human reviews diffs
```

**Best for:** Brownfield projects, small changes, rapid prototyping.

#### 12.3 Autonomous Agent Swarms

```
Human provides goal → Agent swarm designs + builds + tests → Human audits
```

**Best for:** Experimental work, well-bounded tasks with clear evaluation criteria.

All three paradigms benefit from the Three Control Documents. Whether phases are manual or automated, whether a human writes the spec or an orchestrator generates it — the specification, directive, and build log remain the coordination substrate.

---

### 13. The Deeper Shift

We are witnessing the emergence of a new layer in software engineering:

```
Traditional development:
  Human thinks → Human writes code → Human tests → Human ships

Current LLM-assisted development:
  Human thinks → Human writes spec → Agent writes code →
  Agent tests → Human verifies → Human ships

Near-future agentic development:
  Human describes intent → Spec Generator Agent writes spec →
  Builder Agents implement → Verifier Agents test →
  Orchestrator approves → Human audits outcomes

Far-future autonomous development:
  Human describes intent → Agent swarm designs, builds,
  tests, verifies, deploys, monitors → Human audits outcomes
```

At every stage, the Three Control Documents serve the same function: they are the shared protocol enabling coordination between agents. The documents are the invariant. The workflow evolves from manual to semi-automated to fully agentic.

The role of the human shifts from direct implementation to **architectural governance** — defining intent, reviewing outcomes, and ensuring alignment with purpose.

The specification is the contract. The directive is the culture. The build log is the memory.

---

## Appendix A — Anti-Patterns

| Anti-Pattern | Consequence | Prevention |
|---|---|---|
| "Just build it" | Local correctness, global incoherence | Write spec before building |
| Skipping verification | Silent integration bugs | Mandatory verification between phases |
| Modifying spec without logging | Later phases reference outdated design | Spec changelog + Decision Records |
| One giant phase | Focus loss, difficult rollback | 4-8 hour phases maximum |
| Agent self-approval | No check on architectural drift | Human gate between phases |
| Verbose directive (500+ lines) | Agent loses key instructions | Keep under 150 lines |
| Forgetting BUILD_LOG updates | Session recovery fails | Update instruction in every prompt |
| Same model for build + verify | Correlated blind spots | Deterministic checks, CI, review agent |
| Trusting unit tests alone | Cross-boundary bugs survive | Verification tests across phases |
| Committing secrets | Security breach | Pre-commit secret scanning |

---

## Appendix B — Quick-Start Checklist

```
[ ] Write AGENT.md — directive file, < 150 lines
[ ] Write SPEC.md — full technical specification (or modular specs)
[ ] Create BUILD_LOG.md — template with section headers
[ ] Write all phase prompts — one per logical module
[ ] Write all verification prompts — one per phase boundary
[ ] Write ship-readiness verification prompt
[ ] Initialize git with main + develop branches
[ ] Configure pre-commit hooks (linting, secret scanning)
[ ] Create .build/ directory with all control documents
[ ] Symlink AGENT.md to project root
[ ] Identify companion meta-prompts needed (Deployment, Database, etc.)
[ ] Begin Phase 1
```

---

## Appendix C — Measuring Success

| Metric | Target |
|---|---|
| Bugs caught by verification (pre-ship) | > 80% of total bugs |
| Test failures at ship | 0 (non-negotiable) |
| Phases requiring rework after verification | < 20% |
| Session recovery success rate | 100% |
| Spec questions from agent during build | 0 (spec was sufficient) |
| Architectural decisions made by agent | 0 (directive was sufficient) |

**Qualitative indicators:**
- Agent never asks clarifying questions → Spec is complete
- Agent never makes architectural decisions → Directive is complete
- New session picks up seamlessly → Build log is working
- Final product matches spec → Phase protocol is working
- You'd trust a user to install and use it → Ship it

---

*This methodology was developed through practical application, refined through external critique, and published as an open-source meta-prompt. It is itself an artifact of the paradigm it describes: AI execution governed by human architecture, coordinated through shared protocol documents.*

*GitHub: https://github.com/pragnakar/LLM_NATIVE_SOFTWARE_ENGINEERING*

---

## Version History

- v4.0 — Current — Added Naming Conventions section (phases, stages, versions as independent timelines), expanded Meta-Prompt Ecosystem to 11 modules (Security Engineering, MLOps, API Design, Testing Strategy, Documentation, Scrum), updated companion references throughout
- v3.0 — Added Agent Development Runtime (ADR), security model for agentic development, cross-repository knowledge system, full meta-prompt ecosystem section with companion repositories
- v2.0 — Added Phase Protocol (prompt → build → verify → approve), verification evidence table from SAGE build, git strategy with phase tags, pre-commit security section
- v1.0 — Initial publication: Three Control Documents (Directive, Specification, Build Log), .build directory structure, known limitations

---

*Part of the Meta-Prompt Ecosystem: https://github.com/pragnakar*