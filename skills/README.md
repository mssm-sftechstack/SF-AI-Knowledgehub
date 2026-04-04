# SF Skills

Ten core skills covering the Salesforce development patterns that matter most. Each skill is a focused reference guide you can load into any AI tool before coding.

## What a skill is

A skill is a markdown file containing patterns, rules, code examples, and checklists for one area of Salesforce development. You paste it (or reference it) into your AI tool's context and it knows the rules for that area before writing a single line of code.

## Skill list

| Skill | What it covers |
|-------|---------------|
| [sf-apex](sf-apex/) | Trigger framework, service/selector pattern, async patterns, governor limits, security enforcement |
| [sf-lwc](sf-lwc/) | Component structure, LDS-first data access, wire vs imperative, accessibility, security |
| [sf-flow](sf-flow/) | Flow types, fault handling, bypass pattern, order of execution, Transform vs Loop |
| [sf-security](sf-security/) | FLS/CRUD, SOQL injection, XSS, sharing gaps, pre-deploy checklist |
| [sf-test](sf-test/) | 90%+ coverage, bulk testing, runAs pattern, callout mocking, Mixed DML |
| [sf-soql](sf-soql/) | Index usage, selectivity, anti-patterns, relationship queries, WITH SECURITY_ENFORCED |
| [sf-debug](sf-debug/) | Log levels, error root-cause table, Limits tracking, common exceptions |
| [sf-deploy](sf-deploy/) | Deployment order, validate-then-deploy, phase design, CI/CD |
| [sf-permission](sf-permission/) | Permission Sets, FLS enforcement, sharing model, Custom Permissions |
| [sf-integration](sf-integration/) | Named Credentials, callout patterns, inbound REST, idempotency, error handling |

## How to use these with your AI tool

Each skill folder contains two files:
- `README.md` -- plain markdown, works with any AI tool
- `SKILL.md` -- Claude Code format with frontmatter (use this if you're on Claude Code)

See [guides/](guides/) for step-by-step instructions for each tool.

## Quick usage by tool

| Tool | How to load a skill |
|------|-------------------|
| Claude Code | Copy skill folder to `.claude/skills/` in your project, then invoke `/sf-apex` |
| Cursor | Paste the skill's README.md content into `.cursorrules` under a `## SF Apex Rules` header |
| GitHub Copilot | Paste into `.github/copilot-instructions.md` |
| Windsurf | Paste into `.windsurfrules` |
| Any chat-based AI | Paste the README.md content as a system prompt or at the start of your first message |

## Salesforce Order of Execution

The single most important thing to understand before writing triggers, flows, or async code. Every save operation fires in this exact sequence:

1. System validation (required fields, unique fields)
2. **Before-save Flows** (fire BEFORE before-triggers)
3. **Before Apex Triggers**
4. System and custom validation rules
5. Duplicate rules
6. Record saved to database (not committed yet)
7. **After Apex Triggers**
8. Assignment rules, auto-response rules
9. Workflow rules (field updates restart from step 1)
10. Escalation rules
11. **After-save Flows** (fire AFTER after-triggers)
12. Entitlement rules, roll-up summary recalculation
13. Criteria-based sharing evaluation
14. Transaction committed to database

**Critical implications woven into these skills:**
- sf-flow: before-save flows fire at step 2, after-save at step 11
- sf-apex: triggers fire at steps 3 and 7
- sf-integration: callouts are blocked in trigger context (steps 3/7) -- use Queueable
- sf-test: test execution runs the full sequence -- test your bulk behaviour at 200 records

Each skill that touches order-of-execution has a dedicated callout section. Look for the `> Order of Execution Note` blocks.
