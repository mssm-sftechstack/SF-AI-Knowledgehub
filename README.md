<div align="center">

# SF-AI-Knowledgehub

**Exploring AI-assisted Salesforce development with persistent context and memory.**

A collection of patterns, templates, and guardrails for Salesforce developers experimenting with AI tools. Not production-ready. This is a learning project.

[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![Contributions](https://img.shields.io/badge/Contributions-Welcome-orange?style=flat-square)](CONTRIBUTING.md)

</div>

---

## Why This Project

Most Salesforce developers hit the same wall with AI tools. Every new prompt, the AI forgets Salesforce rules. SOQL in loops. Hardcoded IDs. FLS gaps. You debug the same mistakes over and over.

The bigger issue: there's no way to carry lessons from one project to the next. You spend 4 hours debugging a query limit issue in March. In June, someone writes the same pattern. In September, it happens again. Each time you explain it. Each time someone debugs it.

This project asks: what if you could encode those lessons upfront, load them into your AI tool, and catch the mistakes before code review?

---

## What This Project Explores

This isn't about replacing engineers with AI. It's about answering four specific questions:

1. **Can context templates teach guardrails?** If you put Salesforce rules in a CLAUDE.md file, will the AI actually follow them?
2. **What if the AI remembered your past projects?** What would it look like to carry debugging lessons and patterns forward?
3. **Do better prompts actually reduce back-and-forth?** Or does the AI still miss security issues, bulk test failures, and order-of-execution problems?
4. **Can teams share pattern libraries?** What would it take to build a shared library of trigger patterns, SOQL optimizations, and deployment lessons?

The patterns here come from real Salesforce projects (10+ years of implementations). The templates have been tested. But the overall system—how these pieces fit together in an AI workflow—that's still experimental.

---

## What This Isn't

- **Not a deployment tool.** This teaches AI tools how to write correct code, not how to replace architects or senior engineers.
- **Not battle-tested at scale.** These patterns work for the projects they came from. Your org may have different constraints.
- **Not a substitute for code review.** You still need actual humans reviewing security, FLS checks, sharing rules, and deployment order.
- **Not official Salesforce guidance.** These are community patterns. Always verify against [developer.salesforce.com](https://developer.salesforce.com).
- **Memory systems are experimental.** Session memory works within a single project. Cross-project memory is harder and still being figured out.
- **Different AI tools work differently.** A pattern that works in Claude Code might need tweaking for Cursor or Copilot.

For a full breakdown of what's implemented, what's experimental, and what gaps exist, see [LIMITATIONS_AND_STATE.md](LIMITATIONS_AND_STATE.md).

---

## How to Use This Responsibly

This is a learning tool, not production infrastructure.

- **Validate everything manually.** AI makes mistakes. Review the code, test edge cases, check FLS and sharing rules before deploying anywhere.
- **Don't paste production data or credentials.** Keep your org context local. Never share real data in prompts or commit credentials to git.
- **Test in scratch orgs and dev orgs only.** This is for experimenting with workflows, not for live customer systems.
- **Use this to amplify good judgment, not replace it.** When in doubt, ask a senior engineer.

---

## Real Example: Debugging a Query Limit Bug

Your trigger hits 101 queries in production. Here's what happens with and without context.

**Without context files:**
```
Dev: "My trigger queries inside a loop. Fix it."
AI: Rewrites with a map. But forgets WITH SECURITY_ENFORCED.
Result: Code review catches the FLS bug. Two iterations. No bulk test written.
```

**With CLAUDE.md loaded:**
```
AI reads the hard rules:
- Never SOQL or DML inside loops
- Always use WITH SECURITY_ENFORCED  
- Always write a 200-record bulk test

AI rewrites the query, adds WITH SECURITY_ENFORCED, writes the bulk test.
Result: Catches the FLS issue and the query limit before code review. One iteration.
```

That's what this project does. One less round-trip. Fewer surprises in production.

---

## What's Here

| File | What It Covers |
|------|---|
| **[Vibe Coding Salesforce](vibe-coding-salesforce.md)** | How to use AI effectively in Salesforce. Safe patterns, constraints, workflows, component coverage. Start here if you're using Claude Code or other AI tools. |
| **[Getting Started](GETTING_STARTED.md)** | Setup in 5 steps. Scratch orgs, templates, verification. |
| **ARCHITECTURE.md** | Core patterns: 10 rules, layered Apex, bulkification, security, deployment order. |
| **PATTERNS.md** | 8 reusable code examples. Trigger handlers, selectors, tests, LWC, Queueable. |
| **DEPLOYMENT.md** | How to deploy. 16-phase order, CI/CD workflows, GitHub Actions templates. |
| **USE_CASES.md** | Real scenarios. Debugging memory, SOQL optimization, trigger pattern reuse. |
| **LIMITATIONS_AND_STATE.md** | What works, what's experimental, what isn't covered. |
| **templates/** | Ready-to-copy context files (CLAUDE.md, .cursorrules, etc.) |
| **reference/** | Governor limits, deployment order, common errors, metadata types |

---

## The 10 Core Rules

These come from real Salesforce projects. Most are lessons learned the hard way.

1. **No SOQL or DML inside loops.** You'll hit the 101 query limit.
2. **Bulkify for 200+ records.** Dev has 50. Production has 50,000.
3. **Use `with sharing` on every Apex class.** It respects your org's sharing rules.
4. **Use Permission Sets, not Profiles.** Profiles are legacy.
5. **Use `WITH SECURITY_ENFORCED` in every SOQL query.** FLS doesn't enforce itself.
6. **No hardcoded IDs or credentials.** It works in dev. Breaks everywhere else.
7. **One trigger per object.** Extend the handler, never create a second trigger.
8. **No callouts from trigger context.** Use Queueable with Database.AllowsCallouts instead.
9. **90%+ test coverage plus a 200-record bulk test.** Code coverage percentage is misleading without bulk tests.
10. **Validate before you deploy.** It catches surprises.

See [03-architecture-guardrails](03-architecture-guardrails/README.md) for the full reasoning.

---

## How to Use This

**Using AI tools (Claude Code, Cursor, Copilot)?**

1. Read **[Vibe Coding Salesforce](vibe-coding-salesforce.md)** (15 minutes). Learn safe patterns and workflows.
2. Copy [templates/CLAUDE.md](templates/CLAUDE.md) into your project. Load it into your AI tool.
3. Reference [PATTERNS.md](PATTERNS.md) and [ARCHITECTURE.md](ARCHITECTURE.md) in your AI prompts.

**First time here?**

1. Read [GETTING_STARTED.md](GETTING_STARTED.md) (5 minutes). Copy a template into your project.
2. Read [ARCHITECTURE.md](ARCHITECTURE.md) (30 minutes). Understand the 10 core rules and why they matter.
3. Read [USE_CASES.md](USE_CASES.md). See how patterns help in real scenarios.

**Building something?**

1. Look at [PATTERNS.md](PATTERNS.md) for reusable code examples. Copy what fits.
2. Check [ARCHITECTURE.md](ARCHITECTURE.md) for security, bulkification, and testing rules.
3. Use [DEPLOYMENT.md](DEPLOYMENT.md) when you're ready to deploy.

**Reference materials**

- [reference/governor-limits.md](reference/governor-limits.md) — Quick lookup for query limits, DML limits, etc.
- [reference/deployment-order.md](reference/deployment-order.md) — What deploys before what.
- [reference/common-errors.md](reference/common-errors.md) — What went wrong and how to fix it.

**Contributing?**

Found a pattern that works? Security lesson? See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## Who Built This

Sai Shyam (Salesforce Technical Lead, 10+ years of implementations) designed the patterns. Claude helped structure and write the docs.

This is a side project, not an official Salesforce product. It's open source so other teams can learn from it and improve it.

---

## License

MIT. See [LICENSE](LICENSE).

---

> **Not official Salesforce documentation.** Always verify patterns against [developer.salesforce.com](https://developer.salesforce.com). Use at your own discretion in production contexts.
