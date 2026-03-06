# LLM-Native Software Engineering Meta-Prompt

A protocol for coordinating human architects and LLM coding agents in software development through structured control documents, phased build workflows, and mandatory verification discipline.

## What Is a Meta-Prompt?

A meta-prompt is a structured protocol document that initializes an AI-assisted development environment before task execution begins. Instead of starting with an ad-hoc task prompt, the meta-prompt establishes workflow structure, behavioral constraints, verification discipline, and state tracking — so that subsequent prompts operate inside a disciplined build system rather than a blank conversational context.

```
meta-prompt (environment and protocol initialization)
     ↓
task prompt (what to build)
     ↓
AI execution within that structured environment
```

## This Meta-Prompt

LLM-Native Software Engineering is the parent meta-prompt in the ecosystem. It defines:

- **Three Control Documents** — Directive (AGENT.md), Specification (SPEC.md), and Build Log (BUILD_LOG.md) — which serve as the communication protocol between human architects and AI builders
- **The Phase Protocol** — a four-step cycle (prompt → build → verify → approve) that catches integration bugs unit tests cannot detect
- **The .build directory system** — a project-local state machine that evolves from static documents toward an Agent Development Runtime
- **The security model** for agentic development, including agent identity, permission scopes, and secret scanning

This methodology was developed through the construction of SAGE, a production-quality MCP server built entirely with LLM-assisted development across seven phases.

## Part of the Meta-Prompt Ecosystem

This meta-prompt orchestrates companion protocols invoked as needed during the development lifecycle:

| Meta-Prompt | Repository | Purpose |
|---|---|---|
| LLM-Native Software Engineering | https://github.com/pragnakar/LLM_NATIVE_SOFTWARE_ENGINEERING | Architecture and application development |
| Deployment Engineering | https://github.com/pragnakar/Deployment_Engineering | Infrastructure, containerization, and deployment |
| DevOps | https://github.com/pragnakar/DevOps | Runtime operations, observability, and reliability |
| Database | https://github.com/pragnakar/Database | Data persistence, schema design, and management |
| UI/UX | https://github.com/pragnakar/UI-UX | Interface and experience design |

## Usage

1. Load `LLM_NATIVE_SOFTWARE_ENGINEERING.md` into your AI coding agent at the start of a session
2. The agent initializes the development protocol: spec-first design, phased builds, verification discipline, persistent build state
3. Provide the project-specific prompt (what to build)
4. The agent creates control documents (AGENT.md, SPEC.md, BUILD_LOG.md) and begins Phase 1
5. Invoke companion meta-prompts as relevant domains become active

For quick setup, use the Quick-Start Checklist in Appendix B of the meta-prompt document.

## License

Open and collaborative. License information will be defined as the project evolves.

## Author

Pragnakar Pedapenki
