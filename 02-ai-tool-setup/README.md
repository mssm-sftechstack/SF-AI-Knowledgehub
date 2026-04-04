# AI Tool Setup for Salesforce

**Load Salesforce guardrails into your AI tool before writing a single prompt.**

## Why This Matters

AI tools don't know about Salesforce governor limits, the security model, or deployment constraints. Without context, they generate code that:

- Puts SOQL inside loops (hits the 100-query limit instantly on bulk operations)
- Skips field-level security checks (creates audit findings)
- Hardcodes Record Type IDs (breaks in every other org)
- Creates a second trigger on an object (causes duplicate-execution bugs)

Fix this by loading a rules file into your project root. The AI tool picks it up automatically and applies the rules to every suggestion.

## How It Works

Each tool has its own format for the context file. They all encode the same SF rules. The difference is syntax and where the file lives.

| Tool | Context File | Where to Put It | Setup Guide |
|---|---|---|---|
| Claude Code | `CLAUDE.md` | project root | [setup](claude-code/README.md) |
| Cursor | `.cursorrules` | project root | [setup](cursor/README.md) |
| GitHub Copilot | `.github/copilot-instructions.md` | `.github/` folder | [setup](github-copilot/README.md) |
| Windsurf | `.windsurfrules` | project root | [setup](windsurf/README.md) |
| VS Code (other extensions) | `AGENTS.md` | project root | [setup](vscode-general/README.md) |
| Any other tool | `AGENTS.md` | project root | use the [generic template](../templates/AGENTS.md) |

## Templates

All template files live in [`/templates`](../templates/). Copy the one for your tool into your project and fill in your org details where marked.

## Verification

After setup, open your AI tool's chat and ask:

> "What are the Salesforce hard rules for this project?"

It should list back the rules from the file you just added. If it doesn't, the file is in the wrong location or the tool needs a restart.
