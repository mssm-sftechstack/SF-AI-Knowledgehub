# Getting Started

Five minutes to load Salesforce guardrails into your AI tool.

---

## Prerequisites

You need:
- Salesforce CLI (`sf` command): `npm install -g @salesforce/cli`
- VS Code or your preferred editor
- An AI tool: Claude Code, Cursor, GitHub Copilot, or Windsurf
- A Salesforce org (dev org or scratch org)

---

## Step 1: Set Up a Scratch Org (Optional)

If you don't have a dev org, create a scratch org locally.

```bash
# Authenticate your hub org (do this once)
sf org login web --set-default

# Create a scratch org
sf org create scratch --definition-file config/project-scratch-org-sfdx.json \
  --alias my-scratch-org --set-default
```

Now you have a temporary org for testing. It expires in 7 days (or when you delete it).

---

## Step 2: Copy a Template

Pick your AI tool and copy the matching template into your project root:

| Tool | File | Copy to |
|------|------|---------|
| Claude Code | templates/CLAUDE.md | project root |
| Cursor | templates/.cursorrules | project root |
| GitHub Copilot | templates/copilot-instructions.md | `.github/` folder |
| Windsurf | templates/.windsurfrules | project root |
| Any other tool | templates/AGENTS.md | project root |

```bash
# Example: Claude Code
cp templates/CLAUDE.md ./CLAUDE.md
```

Now open the file and fill in your org details where marked:
- Default org alias (e.g., PersSForg_18MAR26)
- API version (current is 62+)
- Instance URL (https://your-instance.my.salesforce.com)

---

## Step 3: Verify It Works

Open your AI tool and ask it to list the Salesforce guardrails:

```
What are the Salesforce hard rules for this project?
```

It should list back the rules from your template file. If it doesn't, the file is in the wrong location or the tool needs a restart.

---

## Step 4: Read the Architecture Guide

See ARCHITECTURE.md for the core patterns:
- The 10 core rules and why they matter
- Layered Apex architecture
- Bulkification and SOQL fundamentals
- Security model (FLS, CRUD, sharing)
- When to use Queueable, Platform Events, or Flows

---

## What's Next

- **Real examples**: See USE_CASES.md for three scenarios showing how memory and patterns help
- **Deployment**: See DEPLOYMENT.md for the 16-phase deployment order and CI/CD setup
- **Reusable patterns**: See PATTERNS.md for trigger handlers, service layers, and query patterns
- **Reference**: See reference/ folder for governor limits, common errors, and metadata types

---

## Common Setup Issues

**"sf command not found"**
Install the CLI globally: `npm install -g @salesforce/cli`

**"AI tool isn't picking up the template"**
- Make sure the file is in the right location (project root, .github/ folder, etc.)
- Restart your AI tool completely
- Some tools cache context; reloading the chat window helps

**"My org details are wrong"**
Open the template file and update the org alias and instance URL. Save. Restart the tool.

**"I don't know what to do next"**
Start with [ARCHITECTURE.md](ARCHITECTURE.md). Read the 10 core rules. Then look at [USE_CASES.md](USE_CASES.md) for real examples of how this helps.
