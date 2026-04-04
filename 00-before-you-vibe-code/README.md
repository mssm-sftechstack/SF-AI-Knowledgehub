# Before You Vibe Code

**Load the right context into your AI tool before writing a single Salesforce prompt.**

---

## What "vibe coding" means in a Salesforce context

Vibe coding means describing what you want in plain English and letting an AI tool generate the code. For a web app, this works reasonably well out of the box. The AI knows JavaScript, React, and REST APIs from millions of public examples.

Salesforce is different. The platform has hundreds of proprietary rules that almost never appear in public code: governor limits, sharing models, deployment ordering, field-level security enforcement, and a metadata structure with no parallel outside Salesforce. An AI tool trained on general code has almost none of this context by default.

The result: the AI writes syntactically correct Apex that fails in production, fails a security review, or won't deploy.

---

## Salesforce is a shared multi-tenant platform

Salesforce hosts over 150,000 customer orgs on shared infrastructure. Every customer runs on the same servers. To prevent one customer's code from degrading everyone else's performance, Salesforce enforces **governor limits** on every transaction.

Governor limits aren't suggestions. They're hard ceilings enforced at runtime:

- 100 SOQL queries per transaction
- 10,000 DML rows per transaction
- 10,000ms CPU time per transaction

An AI tool generating code for a regular web app has no reason to know this. You have to tell it.

---

## The 3 things that go wrong when you vibe code SF without context

### 1. SOQL inside loops

The AI writes a query inside a `for` loop because that's the most natural way to look something up per record. In a small dev org with 5 records, it works fine. In production with 200 records in a single trigger batch, it hits the 101-query limit and throws an unhandled exception for every affected record.

```apex
// **Anti-Pattern** — SOQL inside a loop
for (Opportunity opp : trigger.new) {
    Account acc = [SELECT Name FROM Account WHERE Id = :opp.AccountId];
    // hits limit at 101 records
}
```

### 2. No security enforcement

The AI writes SOQL without `WITH SECURITY_ENFORCED` or a `Security.stripInaccessible()` call. The code returns fields the running user has no access to. This passes testing (admins have access to everything) and fails a security review or exposes data in production.

### 3. Wrong deployment order

The AI generates a Permission Set that references a custom Tab. Tabs must exist in the org before a Permission Set can reference them. Deploying both together in the wrong order fails with a cryptic metadata error. The AI has no way to know this without being told.

---

## The fix: load a context file before prompting

Each AI tool reads a project-level configuration file. Load your Salesforce rules into that file once, and every prompt in that project inherits the context automatically.

| AI Tool | Context file |
|---|---|
| Claude Code | `CLAUDE.md` in project root |
| Cursor | `.cursorrules` in project root |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Windsurf | `.windsurfrules` in project root |

See [02-ai-tool-setup/](../02-ai-tool-setup/) for ready-to-use context file templates for each tool.

---

## Before writing your first prompt

1. Read [multitenant-risks.md](./multitenant-risks.md) to understand what breaks and why.
2. Complete [setup-checklist.md](./setup-checklist.md). 20 items, takes about 10 minutes.
3. Set up your environment using [01-environment-setup/](../01-environment-setup/).
4. Load the context file for your AI tool from [02-ai-tool-setup/](../02-ai-tool-setup/).

Only then: write your first prompt.
