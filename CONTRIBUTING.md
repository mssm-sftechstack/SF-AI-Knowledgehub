# Contributing to SF-AI-Knowledgehub

This repo is community-driven. Every lesson here comes from real implementations, no theory, no fluff.

---

## What We Welcome

- New AI tool context templates (any tool not yet covered)
- Architecture patterns with clear rationale
- Real-world deployment lessons (anonymised, no client names)
- Corrections to existing content
- Mermaid diagrams (render natively in GitHub)
- New sections for areas not yet covered (Einstein, Data Cloud, Industries)

---

## What We Don't Accept

- Content copied verbatim from Salesforce docs (link to it instead)
- Code that violates the 10 guardrails (SOQL in loops, hardcoded IDs, etc.)
- Org-specific or client-specific details
- Promotional content or vendor advertising

---

## How to Contribute

1. Fork the repository
2. Create a branch: `git checkout -b add/your-topic-name`
3. Write your content following the style guide below
4. Submit a pull request with a clear description of what you added and why

---

## Style Guide

### Markdown rules
- H2 (`##`) for major sections, H3 (`###`) for subsections
- Tables for reference material (limits, patterns, comparisons)
- Fenced code blocks with language tag (` ```apex `, ` ```bash `, ` ```xml `)
- Mermaid for flows and architecture (` ```mermaid `)

### Code examples
- All Apex examples must be original, written for this repo
- Use realistic but fictional names (`AccountService`, `WeatherData__c`)
- Comments explain *why*, not just *what*
- Show both wrong and correct patterns where relevant

### Tone
- Direct. Short sentences. No filler.
- Skip phrases like "It is important to", "This ensures that", "Note that"
- Show anti-patterns clearly. Label them **Anti-Pattern** so they're obvious.

---

## Security: No Credentials in Code

Never commit real tokens, keys, passwords, or org IDs. This applies to examples too.

If your example needs to show a credential, use an obviously fake placeholder:

```apex
// WRONG — never put real tokens in code (this is a fake example for illustration only)
req.setHeader('Authorization', 'Bearer mytoken123');

// RIGHT — use Named Credentials
req.setEndpoint('callout:My_Named_Credential/endpoint');
```

To catch accidental credential commits before they happen, install detect-secrets locally:

```bash
pip install detect-secrets
detect-secrets scan > .secrets.baseline
git add .secrets.baseline
```

Run `detect-secrets scan` before every PR. If it flags something, fix it before submitting.

---

## File Naming

- Section folders: `NN-topic-name/` (two-digit prefix for ordering)
- Files: `kebab-case.md`
- Templates: `UPPER-CASE.md` or `.toolname` for dotfiles

---

## Questions?

Open an issue with the `question` label.
