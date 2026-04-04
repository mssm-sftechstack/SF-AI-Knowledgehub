# Using SF Skills with GitHub Copilot

Copilot reads `.github/copilot-instructions.md` as repo-level context for all suggestions.

## Setup

```bash
mkdir -p .github
```

Open the skill's `README.md` and paste relevant sections into `.github/copilot-instructions.md`:

```markdown
# Salesforce Development Rules

## Apex Rules
[paste from skills/sf-apex/README.md -- the rules sections, not the full examples]

## Security Rules
[paste from skills/sf-security/README.md]
```

## What works best with Copilot

Copilot suggestions are influenced by open files as much as instructions. For best results:
- Keep the relevant skill's README.md open in a tab while coding
- Use Copilot Chat (`Ctrl+Shift+I`) with explicit skill context: "Apply the SF apex skill rules to this class"
- The instructions file affects inline completions -- keep it focused on the most critical rules

## Verify

Ask Copilot Chat: "What's the Salesforce security enforcement pattern for SOQL?" It should mention `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()`.
