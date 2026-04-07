# SF-AI-Knowledgehub

**Using AI-assisted development effectively in Salesforce.**

This is a learning project about combining AI coding tools with Salesforce best practices. It's not production code—it's a knowledge base exploring how to get meaningful results from AI when working within Salesforce constraints.

## Why This Exists

If you've used ChatGPT or Claude for Salesforce code, you've probably hit these walls:

- **Lost context between sessions.** You paste the same org details, API version, and governor limits into 20 different prompts. Tedious.
- **Hidden complexity.** AI doesn't see your order of execution, flow quirks, or validation rule side effects unless you spell it out. It confidently generates code that doesn't work in your org.
- **Flows are invisible.** Salesforce Flows are black boxes to AI tools. They don't show up in code, they're not in SOQL results, but they run in production and blow up your logic.
- **Repetitive debugging.** You ask the same questions: "How do I bulkify this?" "What's the FLS pattern?" "Why doesn't my flow trigger?" Same answers, different prompts.
- **Token waste.** You're paying per token. Getting bad code back costs money *and* wastes time.

This project is an experiment in solving those problems.

## What You'll Find Here

15 core documents covering:

- **Order of Execution**: Understanding trigger→flow→validation rule interactions (with diagrams)
- **Flows**: What they do, what they don't, and why AI doesn't see them
- **Security**: FLS, CRUD, stripInaccessible patterns
- **Bulkification**: Why you need it, how to structure for 200+ records
- **Integration patterns**: Callouts, Named Credentials, External Credentials
- **Testing strategy**: Coverage, data factories, assertion patterns
- **SOQL & performance**: Anti-patterns, index usage, selective filters
- **Apex, LWC, Platform Events, Cache**: Hands-on patterns with real code

Each document includes:
- Multi-component perspective (how it works from Apex vs LWC vs Flow)
- Diagrams (sequence diagrams, flowcharts)
- Code examples (current Apex 2025+, valid LWC, real Flow syntax)
- Gotchas and constraints

The goal isn't comprehensive coverage—it's teaching you what to *tell AI* so you get useful code back.

## Real Example: Why Context Matters

You ask your AI tool:

> "How do I query Contact records and update them?"

Without context, you get:
```apex
List<Contact> contacts = [SELECT Id, Name FROM Contact];
for (Contact c : contacts) {
    c.LastName = 'Updated';
    update c;  // ← Loop update, fails at 150 DML limit
}
```

With minimal context ("bulkify for 200+ records, enforce FLS"):

> "I need to query and update Contact records. The org has 150 DML operations per transaction. There are likely 200+ records. Enforce field-level security."

You get:
```apex
List<Contact> contacts = [SELECT Id, Name FROM Contact WHERE IsDeleted = false LIMIT 10000];
List<Contact> toUpdate = new List<Contact>();
for (Contact c : contacts) {
    c.LastName = 'Updated';
    toUpdate.add(c);
}
if (!toUpdate.isEmpty()) {
    Security.stripInaccessible(AccessType.UPDATABLE, toUpdate);
    update toUpdate;
}
```

Same question, 10 seconds of context setup, completely different output.

## Salesforce Constraints You Need to Know

**Order of Execution** (triggers → flows → validation rules):
- A trigger runs and updates a field
- A flow sees that change and fires
- A validation rule sees the flow's changes and blocks the save
- Your logic fails in unexpected places because you didn't know the order

**Flows are not in code**:
- Flows don't appear in `[SELECT ... FROM Contact]` results
- They don't trigger SOQL or DML limits
- But they run after your Apex and can cause cascading failures
- You must *tell AI* about them—they won't find them by reading code

**Bulkification**:
- Not optional—it's a hard constraint
- SOQL limit: 100 queries per transaction
- DML limit: 150 operations per transaction
- Design for at least 200 records or you'll fail in production

**CPU time, heap size, governor limits**:
- 10 seconds of CPU time (synchronous)
- 6 MB heap (synchronous)
- 12 MB heap (asynchronous)
- AI-generated code often ignores these. You can't.

## Security & Responsible Use

**Do not paste production data** (account numbers, customer emails, real records). Ever.

**Be careful with metadata.** Custom field IDs, flow definitions, and permission set assignments can leak org details. Scrub before sharing.

**Always validate AI code before deployment.** AI is a productivity tool, not a replacement for code review. Test with 200+ records. Run security review. Check FLS and CRUD.

**Use this for learning, not copy-paste to prod.** The patterns here are reference implementations. Your org is unique. Adapt, don't copy directly.

## How to Use This Repo

1. **Read the docs** that match your current problem (debugging a trigger? Check Order of Execution. Calling an external API? Check Integration.)
2. **Understand the pattern** — why it works, what constraints matter, what can go wrong
3. **Bring that context to your AI tool.** Don't just ask "How do I...?" — explain your org's limits and constraints
4. **Validate the output.** Test with realistic data. Run security review before deploying

The docs are written for AI too—you can paste relevant sections into your prompts to give context.

## What This Is Not

- Not a production system or framework
- Not Salesforce official documentation (though it references it)
- Not a replacement for code review or testing
- Not "AI coding solved"—it's "here's what we learned about using AI in Salesforce"

## What's Included

### Core Concepts (9 Documents)
- **Order of Execution**: Trigger, flow, validation rule timing and interactions
- **Flow Best Practices**: Types, constraints, recursion prevention, performance
- **Integration Patterns**: Callouts, Named Credentials, retry logic, timeout handling
- **Security Guardrails**: CRUD, FLS, sharing rules, data masking, custom permissions
- **LWC Best Practices**: Lifecycle, state management, error handling, accessibility
- **Testing Strategy**: Unit vs integration, bulk testing, 90%+ coverage patterns
- **Permission Model Design**: Permission Sets, custom permissions, field permissions, role hierarchy
- **AI Pitfalls Matrix**: 12 common mistakes AI makes in Salesforce (with fixes)
- **Multi-Tenant Architecture**: Resource contention, scaling, why limits exist

### Real-World Examples (2 Documents)
- **SOQL Limit Error**: Step-by-step walkthrough of AI without context vs with Salesforce constraints
- **Trigger + Flow Recursion**: Hidden bug scenario, bad code, improved response with context

### Quick Reference (9 Guides)
- Governor limits, SOQL anti-patterns, Batch Apex patterns, LWC security, test data patterns
- SF CLI cheatsheet, common errors, metadata types, deployment order

### Ready-to-Use Templates
- CLAUDE.md for Claude Code
- .cursorrules for Cursor
- copilot-instructions.md for GitHub Copilot
- AGENTS.md for any AI tool

### Getting Started
- Setup in 5 steps
- Architecture overview
- Reusable code patterns
- Deployment playbook
- How to contribute

## Quick Start

**New to this repo?**
1. Read [Order of Execution](ORDER_OF_EXECUTION.md) — understand how Salesforce runs code
2. Read [Architecture](ARCHITECTURE.md) — the 10 core rules and why they matter
3. Copy a [template](templates/) into your project and customize it for your org
4. Reference docs as you build

**Using AI tools (Claude Code, Cursor, Copilot)?**
1. Copy a [template](templates/) into your project
2. Fill in your org details (org alias, API version, instance URL)
3. Ask your AI tool to list Salesforce hard rules — it should cite your template

**Building something specific?**
- Check [Patterns](PATTERNS.md) for code examples
- Check [Real-World Examples](REAL_WORLD_EXAMPLE_SOQL_LIMIT.md) for scenarios
- Browse [Reference](reference/) for quick lookups

## Contributing

Found a pattern that works? Have a lesson from production? See [Contributing](CONTRIBUTING.md).

This is a community project. The more patterns we document, the faster teams can move confidently with AI.

## Who Built This

Based on 10+ years of Salesforce implementation experience. Designed to help developers, teams, and organizations use AI effectively within Salesforce best practices.

Not an official Salesforce product. Open source so others can learn, improve, and share.

## License

MIT. See [LICENSE](LICENSE).

---

> **Not official Salesforce documentation.** Always verify patterns against [developer.salesforce.com](https://developer.salesforce.com). Use at your own discretion in production contexts.
