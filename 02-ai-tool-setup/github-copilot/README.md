# GitHub Copilot Setup for Salesforce

**Copilot reads `.github/copilot-instructions.md` as repo-level context for all suggestions in that repository.**

---

## 1. Enable Copilot in VS Code

1. Install the [GitHub Copilot extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) from the VS Code marketplace.
2. Install the [GitHub Copilot Chat extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) as well.
3. Sign in with your GitHub account when prompted.

## 2. Create the Instructions File

```bash
mkdir -p .github
cp SF-AI-Knowledgehub/templates/copilot-instructions.md .github/copilot-instructions.md
```

The file must be at `.github/copilot-instructions.md` relative to your repo root. Copilot picks it up automatically with no extra configuration.

## 3. Verify the Setup

Open Copilot Chat (`Ctrl+Shift+I` on Windows, `Cmd+Shift+I` on Mac) and ask:

```
What are the Salesforce rules for this repository?
```

Copilot should describe the rules from your instructions file. If it doesn't, check the file is in `.github/` — not `.github/workflows/` or any subdirectory.

## 4. Use Open Files as Additional Context

Copilot also uses your currently open editor tabs as context. Keep relevant files open when working on Salesforce code:

- Keep the related object's `.field-meta.xml` open when writing code that references that field.
- Keep the `TriggerHandler` base class open when writing a new trigger handler.
- Keep the existing test factory class open when writing new test classes.

## 5. Useful Copilot Chat Prompts for Salesforce

After writing an Apex class, try these prompts in Copilot Chat:

```
Review this Apex class against Salesforce multi-tenant governor limit rules.
```

```
Check this SOQL query for missing WITH SECURITY_ENFORCED and index usage.
```

```
Write a test class for this Apex service that covers 90%+ of lines,
includes a 200-record bulk test, and uses System.runAs() for FLS testing.
```

---

## Limitations

| Copilot Feature | Context File Loaded |
|---|---|
| Inline autocomplete | No |
| Copilot Chat (`/ask`, `/explain`) | Yes |
| Copilot Chat (`/fix`, `/tests`) | Yes |
| Copilot Edits (multi-file) | Yes |

Inline autocomplete doesn't load the instructions file. It only uses the current file and open tabs. For reliable rule enforcement, use Copilot Chat rather than accepting autocomplete suggestions blindly.
