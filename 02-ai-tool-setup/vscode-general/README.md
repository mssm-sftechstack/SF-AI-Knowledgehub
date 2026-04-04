# VS Code General AI Setup for Salesforce

**Using VS Code with any AI extension: Continue.dev, Tabnine, Amazon CodeWhisperer, or any chat-based AI tool.**

---

## The Generic Approach

Most AI extensions either read `AGENTS.md` automatically or accept a system prompt at session start. The strategy is the same regardless of which extension you use:

1. Add `AGENTS.md` to your project root with the SF guardrails.
2. Either configure your extension to load it automatically, or paste it as a system prompt at the start of each chat session.

Copy the template:

```bash
cp SF-AI-Knowledgehub/templates/AGENTS.md ./AGENTS.md
```

---

## Extension-Specific Setup

### Continue.dev

Create `.continue/config.json` in your project root:

```json
{
  "models": [
    {
      "title": "Your Model",
      "provider": "your-provider",
      "model": "your-model-name",
      "systemMessage": "You are working on a Salesforce DX project. Read AGENTS.md in the project root for all development rules. Apply every rule without exception."
    }
  ],
  "contextProviders": [
    {
      "name": "file",
      "params": {
        "nRetrieve": 10
      }
    }
  ]
}
```

Then in Continue chat, use `@file AGENTS.md` at the start of complex sessions to load the rules explicitly.

### Tabnine

Tabnine reads your open files for context. Keep `AGENTS.md` open in a tab when working on Salesforce code. For Tabnine Chat, paste the guardrails block below as your first message.

### Amazon CodeWhisperer / Amazon Q

Open the chat panel and paste the guardrails block below as the first message in each session. Amazon Q doesn't currently support a persistent project-level instructions file.

### Any Other Tool

Paste this block at the start of each chat session:

```
You are working on a Salesforce DX project. Apply these rules to every file you write:

- Never put SOQL or DML inside a for loop
- Always use WITH SECURITY_ENFORCED in SOQL queries
- Always use `with sharing` on all Apex classes
- Bulkify all Apex for 200+ records minimum
- Use Permission Sets, not Profiles, for new features
- Never hardcode IDs, Record Type IDs, or credentials
- Resolve Record Type IDs dynamically via getRecordTypeInfosByDeveloperName()
- Never make callouts from trigger context. Use Queueable with Database.AllowsCallouts.
- Every new Apex class needs a paired test class with 90%+ coverage
- Always include a bulk test with 200 records
- Always use @IsTest(SeeAllData=false). Never seeAllData=true.
- Deploy in order: Objects → Fields → Permission Sets → Apex → Triggers → Flows → LWC
- Never deploy without my explicit confirmation
```

---

## Minimum Rules to Paste Every Session

If you can only paste one thing, paste this:

```
Salesforce rules for this session:
1. No SOQL or DML inside for loops
2. WITH SECURITY_ENFORCED on every SOQL query
3. with sharing on every Apex class
4. Bulkify for 200+ records
5. No hardcoded IDs
6. 90%+ test coverage, no seeAllData=true
7. Never deploy without my confirmation
```

These seven rules prevent the most common AI-generated Salesforce bugs.
