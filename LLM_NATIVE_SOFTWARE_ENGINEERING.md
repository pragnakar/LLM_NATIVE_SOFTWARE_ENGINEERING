# LLM-Native Software Engineering

## A Protocol for Coordinating Human and Artificial Intelligence in Software Development

**Version:** 2.0
**Origin:** Derived from the SAGE build (5,861 lines, 459 tests, 7 phases, 0 failures at ship), March 2026
**Author:** Pragnakar Pedapenki
**Applicability:** Claude Code, Cursor, Windsurf, GitHub Copilot, Aider, and any LLM-based coding system — present and future

---

## Part I — The Methodology

---

## 1. What This Document Is

This document defines a repeatable methodology for building production-quality software using LLM coding agents. It was developed through the construction of SAGE (Solver-Augmented Generation Engine) — a non-trivial MCP server built from blank repository to shipped v0.1.0 using a two-instance architecture where a conversational AI designed the system and a coding AI implemented it.

The methodology addresses five problems that emerge when LLMs build software at scale:

| Problem | What Goes Wrong | How This Methodology Prevents It |
|---|---|---|
| Context loss | New sessions start with zero memory | BUILD_LOG.md provides persistent state |
| Architectural drift | Local decisions violate global design | SPEC.md + phase boundaries enforce coherence |
| Silent integration bugs | Unit tests pass, system is broken | Verification prompts test across boundaries |
| Scope creep | Agent "helpfully" adds unrequested features | Phase prompts define bounded scope |
| Session amnesia | Agent repeats or contradicts prior work | Directive + build log restore full context |

This is not theoretical. Every element was tested under real build conditions. Eight ship-blocking bugs were caught exclusively by the verification discipline — none would have been found by unit tests alone.

---

## 2. The Core Insight

LLM coding agents are strong implementers but unreliable architects.

| Role | LLM Performance |
|---|---|
| Architect / system thinker | Weak — inconsistent |
| Implementer | Strong |
| Refactorer | Strong |
| Tester (unit) | Moderate — tends to test happy paths |
| Integration verifier | Weak without explicit prompting |

This asymmetry produces a characteristic failure mode: **local correctness with global incoherence.** The agent writes `db.py`, `cache.py`, and `api.py` — each individually correct, but collectively violating the architecture.

The solution is separation of concerns:

```
┌──────────────────────────────────┐
│  ARCHITECT LAYER                  │
│  (Conversational AI or Human)     │
│                                   │
│  • Designs architecture           │
│  • Writes specifications          │
│  • Creates phase prompts          │
│  • Reviews and approves           │
│  • Makes trade-off decisions      │
│  • Resolves ambiguity             │
└───────────────┬──────────────────┘
                │  Control Documents
                ▼
┌──────────────────────────────────┐
│  BUILDER LAYER                    │
│  (LLM Coding Agent)              │
│                                   │
│  • Reads specifications           │
│  • Implements bounded tasks       │
│  • Writes tests                   │
│  • Updates build log              │
│  • Reports status                 │
│  • Never makes architectural      │
│    decisions                      │
└──────────────────────────────────┘
```

The Architect produces control documents. The Builder executes them. The Builder never decides *what* to build — only *how* to implement what was specified.

---

## 3. The Three Control Documents

Every project requires three control documents. They serve as the communication protocol between architect and builder — analogous to APIs between services, or network protocols between machines. Replace either agent (human or AI) and the protocol still works.

### 3.1 The Directive File — AGENT.md

**Purpose:** Tells the coding agent WHO it is, WHAT it's building, and HOW to behave.

**Naming conventions by platform:**

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
High-level structure. Where code goes. Module responsibilities.

## Tech Stack
Languages, frameworks, libraries, exact versions.

## Build Order
Numbered phase sequence. Agent follows this exactly.

## Critical Design Rules
Non-negotiable constraints. Things the agent must NEVER do.

## Code Style
Type hints, docstrings, naming, logging rules.

## Testing
How to run tests. Known correct values.

## Dependencies
Package names and minimum versions.

## What NOT To Do
Explicit prohibitions preventing architectural violation.
```

**Critical constraint: Keep under 150 lines.** The agent reads this at the start of every session. If it's 500 lines, important instructions get lost in noise. Put details in the SPEC, not the directive.

**Why it matters:** Without this file, every new session makes its own decisions about structure, naming, testing, and error handling. Those decisions are inconsistent across sessions. The directive is the project's culture — it ensures behavioral continuity regardless of which session or model is executing.

### 3.2 The Specification — SPEC.md (or Modular Specs)

**Purpose:** Complete technical specification. The architect's design made explicit and machine-readable-enough for an LLM to implement without clarifying questions.

**For small-to-medium projects (< 10 modules):** A single `SPEC.md` file works.

**For larger projects:** Use modular specifications:

```
.build/spec/
├── architecture.md          ← System-level design
├── data_models.md           ← Shared schemas and types
├── api.md                   ← API contracts
├── modules/
│   ├── solver.md            ← Module-level spec
│   ├── parser.md
│   ├── fileio.md
│   └── server.md
└── schemas/                 ← Machine-readable contracts
    ├── solver_input.json    ← JSON Schema
    ├── api.yaml             ← OpenAPI
    └── config.schema.json
```

**Modular specs solve spec drift.** When the specification is a single large file, it diverges from the codebase after 3-4 phases. With modular specs, each module's specification lives near its implementation and can be updated independently. Machine-readable schemas (JSON Schema, OpenAPI, protobuf) provide an additional layer of truth that can be validated automatically.

**Specification contents per module:**

```markdown
## Module: [name]

### Purpose
What this module does and why it exists.

### Public API
Function signatures with full type hints.

### Data Models
Complete schema definitions (Pydantic, TypeScript interfaces, etc.)

### Behavior
What each function does, step by step.
Input → processing → output, with edge cases.

### Error Handling
What can go wrong. How each error is handled.
Error types, messages, recovery paths.

### Dependencies
What this module imports from other modules.

### Test Cases
Known-correct inputs and expected outputs.
```

**Spec versioning:** When the spec changes during build (it always does), track versions:

```markdown
## Spec Changelog
- v1.1 (Phase 3): Added objective_quadratic to SolverInput for QP support
- v1.2 (Phase 5): Changed relaxation ranking from absolute to percentage-based
```

**Why it matters:** The specification is the handoff mechanism. In the SAGE build, the spec was written by a conversational AI during a design session and read by a coding AI that had never seen the conversation. The spec had to contain everything — because the builder had no other context. Ambiguity in the spec becomes bugs in the code.

### 3.3 The Build Log — BUILD_LOG.md

**Purpose:** Persistent memory across sessions. The coding agent's state file.

**Structure:**

```markdown
# Build Log

## Current Status
- Active Phase: [X]
- Active Branch: [X]
- Last Completed Task: [X]
- Next Task: [X]
- Blockers: [X]

## Session Tracker
| Session | Date | Phase | Duration | Credits | Notes |

## Phase Progress
### Phase 1 — [Name]
- [ ] Task 1
- [ ] Task 2
- [ ] **PHASE 1 COMPLETE**

## Design Decisions
| # | Date | Decision | Rationale |

## Roadblocks & Resolutions
| # | Date | Blocker | Status | Resolution |

## Dependencies
| Package | Version | Status | Notes |

## Test Results
| Phase | Written | Passing | Failing |
```

**The critical instruction:**

> "If BUILD_LOG.md already exists, READ IT FIRST before doing anything. Resume from where the log says you left off. Do not repeat completed work."

This single instruction enables session recovery. During the SAGE build, every session interruption (credit limits, timeouts, context window resets) was recovered seamlessly because the new session read the log and knew exactly where to continue.

**Preventing log bloat:** Over time, the build log grows. When it exceeds ~300 lines, compress completed phases into summaries:

```markdown
### Phase 1 — COMPLETE (Summary)
Project structure, 15 Pydantic schemas, 95 tests passing.
See .build/archive/phase_1_detail.md for full log.
```

Active phase retains full detail. Completed phases are summarized. Full history moves to an archive directory.

---

## 4. The Phase Protocol

Software is built in sequential phases. Each phase follows an identical four-step protocol:

```
PHASE PROMPT ──▶ BUILD ──▶ VERIFY ──▶ APPROVE ──▶ next phase
(architect)      (agent)   (agent)    (human)
```

### 4.1 Phase Prompt

Pre-written by the architect. Specifies:

1. **Git branch:** `feature/phase-N-description`
2. **Exact scope:** Which files to create, which functions to implement
3. **Spec references:** "Implement per SPEC.md Section 3.4" (not repeating the spec content)
4. **Test requirements:** What tests to write, with known-correct values
5. **Commit conventions:** Conventional commit format (`feat:`, `fix:`, `test:`)
6. **Stop condition:** "Do not proceed to Phase N+1 until I review and approve"

**Why pre-written:** Prevents scope creep, ensures consistency, and allows the human to review the plan before committing credits to execution.

### 4.2 Build

The agent executes:
- Creates branch
- Implements code + tests (test-driven or test-alongside)
- Runs tests
- Commits with conventional messages
- Updates BUILD_LOG.md
- Reports completion and stops

### 4.3 Verification Prompt

This is the methodology's most valuable innovation. It is a separate prompt, run after the build, that tests what build-phase tests cannot.

**What verification catches that build tests miss:**

| Build Tests | Verification Tests |
|---|---|
| Does this module work? | Does it work WITH previous modules? |
| Unit correctness | Cross-phase integration |
| Happy path | Error paths and edge cases |
| Schema validity | JSON serialization roundtrips |
| Current phase only | Full pipeline Phase 1 through N |
| Pass/fail | Quality (readable? professional? domain-appropriate?) |

**Structure of a verification prompt:**

```markdown
1. Completeness Check
   Run ALL tests, confirm all functions exist with correct signatures.

2. Output Quality Verification
   Run scenarios, PRINT full output. Check qualitative properties.

3. Cross-Phase Integration Test
   Test the complete pipeline from Phase 1 to N.

4. Error Path Verification
   Test every known failure mode.

5. Next-Phase Readiness
   Verify interfaces the next phase requires.

6. Git Cleanup
   Merge branch, confirm clean state.

7. Update BUILD_LOG.md

8. Structured Report
   Fill in a specific template with pass/fail for each check.
```

**Evidence from the SAGE build — bugs caught exclusively by verification prompts:**

| Phase | Bug | How Found | Impact if Missed |
|---|---|---|---|
| 2 | Binary variable bounds silently ignored | Scheduling integration test | Nurses assigned to unqualified shifts |
| 2 | `float('inf')` breaks JSON serialization | Serialization roundtrip test | Silent data loss in MCP transport |
| 4 | `ffill()` bleeds description rows into data | Messy data integration test | Wrong optimization inputs from Excel |
| 5 | Consecutive days RHS wrong for multi-shift | Cross-phase scheduling pipeline | Feasible models appear infeasible |
| 6 | Console entry point references wrong module | Installation flow simulation | `pip install` works, command fails |
| 7 | Example Excel columns don't match parser | Example smoke test | Every example fails on first try |
| 7 | Blending problem uses % instead of fractions | Example solve test | Example always returns infeasible |
| 7 | pyproject.toml key ordering breaks pip | Installation simulation | Installation fails entirely |

Every one of these passed unit tests. Every one was caught by deliberate cross-boundary verification. This is the strongest argument in the entire methodology.

### 4.4 Approval Gate

The human reviews the verification report and:
- **Approves:** Proceed to next phase
- **Requests fixes:** Agent addresses issues, re-verifies
- **Revises plan:** Architect updates spec or remaining phase prompts

This gate prevents compounding errors. A Phase 3 bug that goes unnoticed becomes a Phase 5 redesign and a Phase 7 rewrite.

---

## 5. Git Strategy

```
main ← merges from develop at milestones only
  └── develop ← integration branch
        ├── feature/phase-1-[name]    → merge to develop after verification
        ├── feature/phase-2-[name]
        └── ...

Tags: phase-N-complete (rollback points), v0.1.0 (releases)
```

**Why this matters for LLM builds:** If Phase 5 corrupts Phase 3's work, you can roll back to `phase-3-complete` and rebuild. Without this, you untangle changes across a flat history — expensive with AI-generated code where diffs are large.

---

## Part II — Known Limitations and Evolution

---

## 6. Where This Methodology Breaks

This methodology was developed on a single-developer, single-project, seven-phase build. It has specific limitations that become apparent at larger scale.

### 6.1 Spec Drift

**Problem:** After 3-4 phases, `SPEC.md` diverges from the codebase. The agent implements against an outdated specification.

**Mitigation (current):** Spec changelog, design decisions log.

**Better solution (future):** Machine-readable specification artifacts that stay synchronized with code:
- JSON Schema files validated in CI
- OpenAPI specs generated from code
- TypeScript interfaces or protobuf definitions as source of truth
- Modular specs per module, updated as part of the phase that modifies them

The principle: **parts of the spec should be executable, not just readable.**

### 6.2 Waterfall Rigidity

**Problem:** The methodology is sequential. Real systems require refactoring, backtracking, and mid-build redesign. LLMs often discover problems during implementation that invalidate earlier phases.

**Mitigation (current):** Verification prompts catch integration issues before they compound.

**Better solution (future):** An iterative loop:

```
spec → build → verify → revise spec → continue
```

Phase prompts should anticipate revision. When verification reveals a spec deficiency, the protocol should be: update the spec, log the decision, adjust downstream phases — not force through the original plan.

### 6.3 Architect Bottleneck

**Problem:** The methodology depends on one human architect writing specs, prompts, and verification criteria. For large systems, this becomes a bottleneck — the architect spends more time writing specs than reviewing code.

**Mitigation (current):** Pre-written phase and verification prompts reduce per-phase architect effort.

**Better solution (future):** Spec fragments per module rather than monolithic specs. Architect defines system-level architecture and interfaces; module-level specs can be generated or co-authored by the coding agent under architectural constraints. Eventually, an orchestrator agent replaces the human architect for routine decisions.

### 6.4 Verification Is Still LLM-Dependent

**Problem:** Even verification prompts are executed by an LLM. Builder mistakes and verifier mistakes can correlate — the same model may have the same blind spots in both roles.

**Mitigation (current):** Verification prompts include deterministic checks (assert statements, JSON roundtrips, file existence checks) alongside qualitative assessment.

**Better solution (future):** A layered verification stack:

| Layer | What It Catches | Tool |
|---|---|---|
| Static analysis | Type errors, style violations | mypy, ruff, eslint |
| Schema validation | Interface contract violations | JSON Schema, OpenAPI validators |
| Unit tests | Module-level correctness | pytest, jest |
| Integration tests | Cross-module interaction bugs | Verification prompts |
| Deterministic CI | Regression, installation, packaging | GitHub Actions, CI pipeline |
| LLM review agent | Spec compliance, architectural drift | Separate model instance |

Verification prompts should progressively become CI jobs. The SAGE build used manual prompts; a mature workflow automates the mechanical checks and reserves LLM verification for qualitative assessment only.

### 6.5 Build Log Bloat

**Problem:** BUILD_LOG.md grows with every session. Eventually the agent's context window can't hold it, and it stops reading the full log.

**Solution:** Structured log with active summarization:

```
.build/
├── BUILD_STATE.md         ← Current status only (< 50 lines)
├── BUILD_SUMMARY.md       ← Phase summaries (< 200 lines)
└── archive/
    ├── phase_1_log.md     ← Full detail, completed phases
    ├── phase_2_log.md
    └── ...
```

The agent reads `BUILD_STATE.md` (tiny, always current) and `BUILD_SUMMARY.md` (compressed history). Full logs exist for human review but aren't loaded into agent context.

---

## Part III — The Future Architecture

---

## 7. Evolution of the .build System

### 7.1 Current State: Static Documents

Today, `.build/` contains markdown files that humans paste into agent prompts:

```
.build/
├── AGENT.md
├── SPEC.md
├── BUILD_LOG.md
├── PHASE_PROMPTS.md
└── VERIFICATION_PROMPTS.md
```

This works for single-developer, single-project builds. It was sufficient for SAGE. But it will not scale.

### 7.2 Near-Term Evolution: Active Tooling

The next step is executable tools alongside documents:

```
.build/
├── docs/
│   ├── AGENT.md
│   ├── spec/
│   │   ├── architecture.md
│   │   ├── data_models.md
│   │   └── modules/
│   └── decisions/                   ← Architectural Decision Records
│       ├── ADR-001-solver-choice.md
│       ├── ADR-002-file-format.md
│       └── ADR-003-api-design.md
│
├── state/
│   ├── BUILD_STATE.md               ← Current status (< 50 lines)
│   ├── BUILD_SUMMARY.md             ← Compressed history
│   └── archive/                     ← Full phase logs
│
├── prompts/
│   ├── phases/
│   │   ├── phase_1.md
│   │   └── ...
│   ├── verification/
│   │   ├── verify_phase_1.md
│   │   └── ...
│   └── templates/                   ← Reusable prompt templates
│       ├── phase_prompt_template.md
│       └── verification_template.md
│
├── schemas/                          ← Machine-readable contracts
│   ├── solver_input.json
│   ├── api.yaml
│   └── config.schema.json
│
├── scripts/
│   ├── verify_phase.py              ← Automated verification runner
│   ├── summarize_log.py             ← Log compression tool
│   └── validate_schemas.py          ← Schema compliance checker
│
├── eval/
│   ├── benchmark_problems/           ← Known-answer test cases
│   └── quality_rubrics.md            ← Evaluation criteria
│
└── telemetry/
    ├── token_usage.csv               ← Cost tracking per phase
    └── failure_modes.md              ← Patterns of agent mistakes
```

**Architectural Decision Records (ADRs)** deserve special mention. Large projects benefit from recording design decisions in a structured format. When a new agent session encounters code and wonders "why was it built this way?", ADRs provide the historical reasoning that the code itself cannot express. Both human developers and AI agents benefit from this context.

**Agent telemetry** tracks how agents perform across phases: token usage, failure modes, retry patterns, verification failure types. This data optimizes prompts, workflows, and agent configurations over time. It transforms .build from a control system into a learning system.

### 7.3 Medium-Term Evolution: Externalized State

As projects scale, markdown files in a repository become insufficient. The `.build` layer evolves to externalize its state:

**Externalized Project Memory:**
BUILD_LOG.md transitions from a flat file into a structured interface to a persistent storage system (NoSQL database, knowledge graph, or specialized build state service). The repository contains only pointers and connection configurations:

```
.build/
├── config.yaml                      ← Connection to external build state
│   # build_state_uri: https://buildstate.example.com/project/sage
│   # auth: vault://build-state-token
│
├── .cache/
│   └── latest_state.md              ← Cached summary for offline work
│
└── schemas/                          ← Still local (source of truth)
```

All persistent workflow data — build logs, phase history, verification results, specification versions, agent execution traces — resides in the centralized store. The repository remains focused on source code, tests, documentation, and configuration.

**Versioned Specification Registry:**
SPEC.md evolves into a versioned specification stored externally, with the repository containing only the current working interface. This enables:
- Automated diffing between architecture revisions
- Traceability between code commits and spec changes
- Rollback to previous architectural states
- Spec version pinning per branch

**Why cloud-hosted build state matters:**
- Prevents repository bloat
- Enables real-time collaboration between developers and agents
- Supports scalable querying of build history and agent activity
- Allows centralized governance for large organizations
- Enables integration with CI/CD and project management tools

Under this design, the local `.build` directory becomes a gateway to a distributed build orchestration system.

### 7.4 Integration with Agile Planning Systems

To support team-scale development, the `.build` system exposes integration layers to common planning tools:

| Planning Tool | Integration |
|---|---|
| Jira / Linear / ClickUp | Phase prompts ↔ sprint tickets (bi-directional) |
| Notion | Spec sections ↔ documentation pages |
| GitHub Issues | Verification failures → auto-created issues |
| Slack / Teams | Phase completion → notifications |
| CI/CD | Verification prompts → automated pipeline stages |

Phase prompts and verification checkpoints automatically map to tickets, sprint tasks, or milestones. Changes in planning tools update development specifications. Agent execution results update project dashboards in real time.

This connects agent-driven development with traditional agile planning, ensuring AI-assisted development remains visible and manageable within standard organizational processes.

### 7.5 Long-Term Vision: Agentic Development Infrastructure

The logical endpoint is a multi-agent orchestration system where the human provides intent and agents coordinate the full development lifecycle:

```
┌─────────────────────────────────────────────────────┐
│  ORCHESTRATOR AGENT                                  │
│  Reads spec, generates phases, dispatches builders,  │
│  runs verification, approves/rejects, manages git    │
└──────────┬──────────────────┬───────────────────────┘
           │                  │
    ┌──────▼──────┐    ┌──────▼──────┐
    │ BUILDER     │    │ BUILDER     │
    │ AGENT A     │    │ AGENT B     │
    │ (solver)    │    │ (file I/O)  │    ← parallel where
    └──────┬──────┘    └──────┬──────┘      dependencies allow
           │                  │
    ┌──────▼──────┐    ┌──────▼──────┐
    │ REVIEW      │    │ REVIEW      │
    │ AGENT A     │    │ AGENT B     │    ← adversarial check
    └──────┬──────┘    └──────┬──────┘
           │                  │
    ┌──────▼──────────────────▼──────┐
    │  INTEGRATION VERIFIER           │
    │  Tests cross-module coherence   │
    └────────────────┬────────────────┘
                     │
              ┌──────▼──────┐
              │   HUMAN     │
              │   AUDITOR   │    ← reviews outcomes, not process
              └─────────────┘
```

In this model:
- The **Orchestrator** replaces the human architect for routine decisions
- **Builder Agents** execute phases in parallel where dependency graphs allow
- **Review Agents** provide adversarial checks using a different model or temperature, reducing correlated failures
- **Integration Verifiers** run deterministic CI plus LLM-based coherence checks
- The **Human** shifts from directing every step to auditing outcomes
- The **Three Control Documents remain the coordination protocol** — they are the invariant across all levels of automation

**Multi-agent orchestration file:**

```markdown
# ORCHESTRATION.md

## Agent Roles
- Builder A: core library (Phases 1-5)
- Builder B: server layer (Phase 6, after Phase 5 gate)
- Verifier: runs after each phase completion

## Dependency Graph
Phase 3 depends on: Phase 1, Phase 2
Phase 4 depends on: Phase 1 (not Phase 2 or 3)
Phase 5 depends on: Phase 2, Phase 3, Phase 4
Phase 6 depends on: Phase 5

## Parallel Opportunities
Phase 3 (builder) and Phase 4 (fileio) can run in parallel
after Phase 2 — they share Phase 1 schemas but don't
depend on each other.
```

### 7.6 Build State Snapshots

A powerful concept for collaborative agentic development: Git-like commits for the entire development workflow state, not just code.

A **Build State Snapshot** captures:
- Current specification version
- Build log state
- Phase completion status
- All architectural decisions made
- Verification results
- Agent configuration and prompt versions

This enables:
- Branching development workflows (not just code)
- Reverting to a previous workflow state when an approach fails
- Comparing how different agent configurations perform on the same project
- Reproducible builds where the entire context — not just the code — is versioned

### 7.7 Cross-Repository Knowledge

Organizations working on multiple projects benefit from a shared architectural knowledge base:

```
org-build-knowledge/
├── patterns/
│   ├── mcp-server-pattern.md
│   ├── api-service-pattern.md
│   └── data-pipeline-pattern.md
├── lessons/
│   ├── highs-solver-quirks.md
│   └── mcp-sdk-gotchas.md
├── templates/
│   ├── python-monorepo/
│   └── typescript-api/
└── metrics/
    └── agent-performance-baselines.csv
```

Agents working across projects access common architectural knowledge, reducing duplication and enabling consistent engineering practices. This transforms `.build` from a per-project tool into an organizational memory system for software development.

---

## Part IV — The Paradigm Shift

---

## 8. Three Emerging Paradigms

The industry is exploring three distinct approaches to AI-assisted development. They haven't converged yet.

### 8.1 Spec-Driven Development (This Methodology)

```
Human writes spec → Agent implements → Human verifies → Iterate
```

**Strengths:** High coherence, predictable outcomes, clear accountability.
**Weaknesses:** Architect bottleneck, spec maintenance overhead, waterfall tendency.
**Best for:** Greenfield projects, well-understood domains, production-grade output.

### 8.2 Repo-Evolution (Cursor / Windsurf Style)

```
Human has existing codebase → Agent modifies in-place → Human reviews diffs
```

**Strengths:** Low overhead, works with existing code, rapid iteration.
**Weaknesses:** Architectural drift over time, inconsistent quality, no global coherence guarantee.
**Best for:** Brownfield projects, small changes, rapid prototyping.

### 8.3 Autonomous Agent Swarms

```
Human provides high-level goal → Agent swarm designs + builds + tests → Human audits
```

**Strengths:** Maximum autonomy, potential for parallel execution, minimal human involvement.
**Weaknesses:** Unpredictable, difficult to debug, prone to compounding errors without strong coordination protocols.
**Best for:** Experimental, research, well-bounded tasks with clear evaluation criteria.

**The key insight:** All three paradigms benefit from the Three Control Documents. Whether a human writes the spec or an orchestrator agent generates it, whether phases are manual or automated, whether verification is prompted or CI-driven — the specification, directive, and build log remain the communication substrate that enables coordination.

---

## 9. The Deeper Shift

This document quietly demonstrates something larger than a development methodology:

**Software development with LLMs is becoming a protocol problem, not a coding problem.**

The Three Control Documents function as a communication protocol between agents — human or artificial. This is analogous to:
- APIs between services
- Network protocols between machines
- Distributed systems coordination patterns

We are moving from:
```
Human coding + AI assistance
```
To:
```
AI execution + human architecture
```
And eventually toward:
```
Multi-agent coordination governed by shared protocols
```

The documents are the invariant. The workflow evolves from manual to semi-automated to fully agentic. But the need to specify before building, to verify before proceeding, and to remember across sessions remains as long as complex software needs to be built correctly.

The specification is the contract. The directive is the culture. The build log is the memory.

---

## 10. Anti-Patterns

| Anti-Pattern | What Goes Wrong | Prevention |
|---|---|---|
| "Just build it" | Local correctness, global incoherence | Write spec before building |
| Skipping verification | Silent integration bugs accumulate | Mandatory verification between phases |
| Modifying spec without logging | Later phases reference outdated design | Spec changelog + Design Decisions Log |
| One giant phase | Focus loss, large uncommitted changes, difficult rollback | 4-8 hour phases maximum |
| Agent self-approval | No external check on architectural drift | Human gate between phases |
| Verbose directive (500+ lines) | Agent loses important instructions in noise | Keep under 150 lines |
| Forgetting BUILD_LOG updates | Session recovery fails | Update instruction in every phase prompt |
| Trusting unit tests alone | Cross-boundary bugs survive to production | Verification tests across phases |
| Same model for build + verify | Correlated failures (same blind spots) | Deterministic checks, different model for review, CI |

---

## 11. Quick-Start Checklist

```
[ ] Write AGENT.md — directive file, < 150 lines
[ ] Write SPEC.md — full technical specification (or modular specs)
[ ] Create BUILD_LOG.md — template with section headers
[ ] Write all phase prompts — one per logical module
[ ] Write all verification prompts — one per phase boundary
[ ] Write ship-readiness verification prompt
[ ] Initialize git with main + develop branches
[ ] Create .build/ directory with all control documents
[ ] Symlink AGENT.md to project root
[ ] Begin Phase 1
```

---

## 12. Measuring Success

| Metric | SAGE v0.1.0 | Target |
|---|---|---|
| Bugs caught by verification (pre-ship) | 8 | > 80% of total bugs |
| Tests at ship | 459 | Proportional to codebase |
| Test failures at ship | 0 | 0 (non-negotiable) |
| Phases requiring rework after verification | 0 | < 20% |
| Session recovery success rate | 100% | 100% |
| Spec questions from agent during build | 0 | 0 (spec was sufficient) |
| Architectural decisions made by agent | 0 | 0 (directive was sufficient) |

**Qualitative indicators:**
- Agent never asks clarifying questions → Spec is complete
- Agent never makes architectural decisions → Directive is complete
- New session picks up seamlessly → Build log is working
- Final product matches spec → Phase protocol is working
- You'd trust a user to install and use it → Ship it

---

*This methodology was developed through the construction of SAGE v0.1.0 (https://github.com/pragnakar/Project_Sage) — 5,861 lines, 459 tests, 7 phases, 33 commits, 8 bugs caught by verification, zero failures at ship. It is itself an artifact of the paradigm it describes: AI execution governed by human architecture, coordinated through shared protocol documents.*
