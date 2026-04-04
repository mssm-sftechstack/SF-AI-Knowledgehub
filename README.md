# SF-AI-Knowledgehub

> **Vibe code Salesforce — with architecture guardrails baked in.**

A community playbook for Salesforce developers and architects who use AI coding tools (Claude Code, Cursor, GitHub Copilot, Windsurf, and others). One repo to set the right context before you write your first prompt — so your AI tool respects Salesforce's multi-tenant architecture, governor limits, security model, and deployment patterns automatically.

---

## Why This Exists

AI tools are transforming how Salesforce is built. But Salesforce runs on a **shared multi-tenant platform** with strict resource limits, a layered security model, and a deployment pipeline that has real order-of-operations constraints. Most AI tools don't know this — unless you tell them.

This repo gives you:
- **Ready-to-use context files** for every major AI tool (Claude Code, Cursor, Copilot, Windsurf)
- **Architecture guardrails** — patterns that prevent the most common multi-tenant mistakes
- **Security model guidance** — FLS, sharing, permission sets explained clearly
- **Agentforce patterns** — how to build AI agents on Salesforce correctly
- **DevOps playbooks** — scratch orgs, CI/CD, deployment order

> Not official Salesforce documentation. This is community knowledge — real lessons from 10+ years of Salesforce implementations, distilled into AI-ready context.

---

## Who This Is For

| Person | What You Get |
|--------|-------------|
| SF Developer new to AI tools | Step-by-step setup so your AI tool knows SF rules before it codes |
| Experienced SF Dev | Architecture patterns and guardrail templates to prevent regressions |
| Architect | Multi-tenant design principles, security model, Agentforce architecture |
| DevOps Engineer | CI/CD patterns, deployment order, scratch org workflows |
| Anyone "vibe coding" Salesforce | The context file that stops your AI from writing dangerous SF code |

---

## Quick Start — Load Guardrails in 2 Minutes

Pick your AI tool and copy the matching context file into your Salesforce project root:

| Tool | File to Copy | Path |
|------|-------------|------|
| Claude Code | [CLAUDE.md](templates/CLAUDE.md) | project root |
| Cursor | [.cursorrules](templates/.cursorrules) | project root |
| GitHub Copilot | [copilot-instructions.md](templates/copilot-instructions.md) | `.github/` folder |
| Windsurf | [.windsurfrules](templates/.windsurfrules) | project root |
| Any tool | [AGENTS.md](templates/AGENTS.md) | project root |

That's it. Your AI tool now knows:
- Never put SOQL or DML inside a for loop
- Always use Permission Sets, not Profiles, for new features
- Never hardcode IDs or credentials
- Always bulkify for 200+ records
- Use `with sharing` on every Apex class
- The correct deployment order for complex changes

---

## Repository Structure

```
SF-AI-Knowledgehub/
├── 00-before-you-vibe-code/    ← Start here. Why context matters.
├── 01-environment-setup/       ← SF CLI, VS Code, scratch orgs
├── 02-ai-tool-setup/           ← Claude Code, Cursor, Copilot, Windsurf config
├── 03-architecture-guardrails/ ← Multi-tenant, order of execution, async patterns
├── 04-security-model/          ← Permission sets, sharing, FLS/CRUD
├── 05-development-patterns/    ← Apex, LWC, Flow, Integration patterns
├── 06-agentforce/              ← AI agent design on Salesforce
├── 07-devops/                  ← CI/CD, scratch orgs, deployment strategy
├── templates/                  ← Copy-paste context files for each AI tool
└── reference/                  ← Governor limits, deployment order, errors
```

---

## The Guardrails (TL;DR)

These are the non-negotiable rules encoded into every context template:

1. No SOQL or DML inside any loop
2. Bulkify everything — design for 200+ records minimum
3. Use `with sharing` on every Apex class
4. Use Permission Sets for all new features — never Profiles
5. Use `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()` in every SOQL query
6. Never hardcode IDs, Record Type IDs, or credentials
7. One trigger per object — extend the handler, never create a second trigger
8. Never make callouts from trigger context — use Queueable with `Database.AllowsCallouts`
9. Test classes must have 90%+ coverage, bulk test (200 records), and `@IsTest(SeeAllData=false)`
10. Always validate and dry-run before deploying

Full rationale for each rule: [03-architecture-guardrails](03-architecture-guardrails/README.md)

---

## Contributing

This repo grows with the community. See [CONTRIBUTING.md](CONTRIBUTING.md) to:
- Add patterns for a tool we haven't covered
- Submit a new architecture guardrail with explanation
- Share a real-world deployment lesson
- Improve existing documentation

---

## Author

Built and maintained by a Salesforce Technical Lead with 10+ years of experience building enterprise implementations. Passionate about the intersection of Salesforce architecture and AI-assisted development.

- GitHub: [@mssm-sftechstack](https://github.com/mssm-sftechstack)
- LinkedIn: [Sai Shyam Meda](https://www.linkedin.com/in/meda-sai-shyam/)

---

## License

MIT — use freely, contribute back.

---

## Disclaimer

This is independent community knowledge, not official Salesforce documentation. Always refer to [developer.salesforce.com](https://developer.salesforce.com) for authoritative guidance. All code examples are original and written for educational purposes.
