<div align="center">

# SF-AI-Knowledgehub

**SF architecture playbook for AI-assisted development.**
Context templates for Claude Code, Cursor, Copilot, and Windsurf with multi-tenant guardrails, security model, Agentforce patterns, and DevOps workflows baked in.

[![Files](https://img.shields.io/badge/Files-54-blue?style=flat-square)](#whats-inside)
[![Sections](https://img.shields.io/badge/Sections-9-green?style=flat-square)](#whats-inside)
[![AI Tools](https://img.shields.io/badge/AI_Tools-5-purple?style=flat-square)](#quick-start)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![Contributions](https://img.shields.io/badge/Contributions-Welcome-orange?style=flat-square)](CONTRIBUTING.md)

</div>

---

## The Problem

Your AI tool doesn't know Salesforce. It'll write SOQL in loops, skip FLS checks, hardcode IDs, and deploy in the wrong order. Copy one file from this repo and it knows the rules before prompt one.

---

## Quick Start

Pick your tool and copy the matching file into your project:

| Tool | File to copy | Where it goes |
|------|-------------|---------------|
| Claude Code | [CLAUDE.md](templates/CLAUDE.md) | project root |
| Cursor | [.cursorrules](templates/.cursorrules) | project root |
| GitHub Copilot | [copilot-instructions.md](templates/copilot-instructions.md) | `.github/` folder |
| Windsurf | [.windsurfrules](templates/.windsurfrules) | project root |
| Any tool | [AGENTS.md](templates/AGENTS.md) | project root |

```bash
# Example: Claude Code setup
cp templates/CLAUDE.md /path/to/your/sf-project/CLAUDE.md
```

---

## How Vibe Coding Salesforce Works

Most developers jump straight to prompting. That's where things go wrong. The diagram below shows the right flow — context first, then code.

```mermaid
flowchart TD
    DEV([👨‍💻 You]) --> IDEA[Feature Idea]

    IDEA --> CTX["Step 1: Load Context
    Copy one file into your project root
    CLAUDE.md · .cursorrules · .windsurfrules · AGENTS.md"]

    CTX --> TOOL["Step 2: Pick Your AI Tool
    Claude Code · Cursor · GitHub Copilot · Windsurf"]

    TOOL --> DESIGN["Step 3: Design First
    Ask AI to produce a spec before any code
    — What files? What layers? What deploy order?"]

    DESIGN --> GUARD{"Guardrails Active?"}

    GUARD -- "No context loaded" --> BAD["❌ AI writes bad SF code
    SOQL in loops · hardcoded IDs
    No FLS · wrong deploy order
    Works in dev, breaks in prod"]

    GUARD -- "Context loaded ✅" --> GOOD["✅ AI writes correct SF code
    Bulkified · FLS enforced
    Layered architecture
    Correct deploy order"]

    GOOD --> LAYERS["Step 4: Layered Apex Architecture
    Trigger → TriggerHandler → Service → Selector"]

    LAYERS --> SEC["Step 5: Security Check
    WITH SECURITY_ENFORCED in SOQL
    with sharing on every class
    Permission Sets · FLS · Sharing Rules"]

    SEC --> TEST["Step 6: Test
    90%+ coverage · 200-record bulk test
    System.runAs() for FLS tests
    Mock callouts"]

    TEST --> DEPLOY["Step 7: Deploy Right
    Objects → Permission Sets → Apex
    → Triggers → Flows → LWC → FlexiPages"]

    DEPLOY --> PROD([🚀 Production])

    style BAD fill:#ff6b6b,color:#fff
    style GOOD fill:#51cf66,color:#fff
    style PROD fill:#339af0,color:#fff
    style CTX fill:#845ef7,color:#fff
    style GUARD fill:#fd7e14,color:#fff
```

The left path (no context) is where most vibe coding disasters happen. The right path is what this repo enables.

---

## What's Inside

| Section | What you get |
|---------|-------------|
| 00 Before You Vibe Code | Why context matters, multi-tenant risks, pre-flight checklist |
| 01 Environment Setup | Prerequisites, org types, SF CLI, VS Code, scratch orgs |
| 02 AI Tool Setup | Claude Code, Cursor, Copilot, Windsurf config guides |
| 03 Architecture Guardrails | Multi-tenant, layered Apex, order of execution, async, SOQL |
| 04 Security Model | Permission Sets, FLS/CRUD, sharing model |
| 05 Development Patterns | Apex, LWC, Flow, Integration patterns |
| 06 Agentforce | Topics, Actions, InvocableApex, Trust Layer |
| 07 DevOps | Deployment strategy, GitHub Actions CI/CD |
| Templates | Ready-to-copy context files for every AI tool |
| Reference | Governor limits, deployment order, common errors, metadata types |

---

## The 10 Guardrails

1. **No SOQL or DML inside loops**: hits the 101 query limit at 101 records
2. **Bulkify for 200+ records**: dev orgs have tiny data, production doesn't
3. **`with sharing` on every Apex class**: respects record visibility rules
4. **Permission Sets, not Profiles**: Profiles are legacy, PS is the roadmap
5. **`WITH SECURITY_ENFORCED` in every SOQL query**: FLS doesn't enforce itself
6. **No hardcoded IDs or credentials**: works in dev, breaks in every other org
7. **One trigger per object**: extend the handler, never create a second trigger
8. **No callouts from trigger context**: use Queueable with Database.AllowsCallouts
9. **90%+ test coverage, 200-record bulk test**: coverage numbers lie without bulk tests
10. **Validate before you deploy**: dry-run first, always

Full rationale: [03-architecture-guardrails](03-architecture-guardrails/README.md)

---

## Built By

Co-built by **[Sai Shyam](https://www.linkedin.com/in/meda-sai-shyam/)** and Claude.

Sai Shyam is a Salesforce Technical Lead with 10+ years of enterprise implementations. He contributed the architecture patterns, security model, deployment lessons, and Agentforce patterns that form the backbone of this repo. Claude helped structure and write the documentation.

[![GitHub](https://img.shields.io/badge/GitHub-mssm--sftechstack-24292e?style=flat-square&logo=github&logoColor=white)](https://github.com/mssm-sftechstack)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sai_Shyam-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/meda-sai-shyam/)

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) to add patterns, share deployment lessons, or improve existing content.

---

## License

MIT. See [LICENSE](LICENSE).

---

> Independent community knowledge, not official Salesforce documentation. Always verify against [developer.salesforce.com](https://developer.salesforce.com).
