# Contributing to SF-AI-Knowledgehub

This repo is community-driven — every lesson, pattern, and guardrail shared here helps someone avoid a costly mistake on a live Salesforce org.

---

## What We Welcome

- New AI tool context templates (any tool not yet covered)
- Architecture patterns with clear rationale
- Real-world deployment lessons (anonymised — no client names)
- Corrections to existing content
- Diagrams (Mermaid preferred — renders in GitHub)
- New sections for areas not yet covered (Einstein, Data Cloud, Industries)

## What We Don't Accept

- Content copied verbatim from Salesforce documentation (link to it instead)
- Code that violates the guardrails (SOQL in loops, hardcoded IDs, etc.)
- Org-specific or client-specific details
- Promotional content or vendor-specific tool advertising

---

## How to Contribute

1. Fork the repository
2. Create a branch: `git checkout -b add/your-topic-name`
3. Write your content following the style guide below
4. Submit a pull request with a clear description of what you added and why

---

## Style Guide

### Markdown
- Use H2 (`##`) for major sections, H3 (`###`) for subsections
- Tables for reference material (limits, patterns, comparisons)
- Fenced code blocks with language tag (` ```apex `, ` ```bash `, ` ```xml `)
- Mermaid diagrams for flows and architecture (` ```mermaid `)

### Code Examples
- All Apex examples must be original — written for this repo, not copied
- Use realistic but fictional names (AccountService, WeatherData__c, etc.)
- Include comments that explain *why*, not just *what*
- Show both the wrong pattern and the correct pattern where relevant

### Tone
- Plain English — this repo is for everyone from junior devs to architects
- No jargon without explanation
- No AI-generated filler ("It is important to note that...", "This ensures that...")
- Short sentences. Tables beat paragraphs for reference material.

### Guardrail Rule
Every code example must follow the 10 core guardrails in the README. If you submit code that violates them (even as a "wrong" example), wrap it in a clearly labelled **Anti-Pattern** block.

---

## File Naming

- Section folders: `NN-topic-name/` (two-digit prefix for ordering)
- Files: `kebab-case.md`
- Templates: `UPPER-CASE.md` or `.toolname` (for dotfiles)

---

## Questions?

Open an issue with the `question` label. We aim to respond within a few days.
