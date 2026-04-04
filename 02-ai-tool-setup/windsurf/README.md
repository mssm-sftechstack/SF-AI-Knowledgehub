# Windsurf Setup for Salesforce

**Windsurf (by Codeium) reads `.windsurfrules` from your project root as persistent context for Cascade, its AI agent.**

---

## 1. Install Windsurf

Download from [codeium.com/windsurf](https://codeium.com/windsurf) and install. Windsurf is a VS Code fork with Cascade — an agentic AI that can read, write, and execute across multiple files in one session.

## 2. Open Your Salesforce Project

```
File → Open Folder → select your SFDX project root
```

The project root is the folder containing `sfdx-project.json`.

## 3. Add the Rules File

```bash
cp SF-AI-Knowledgehub/templates/.windsurfrules ./.windsurfrules
```

The file must be named exactly `.windsurfrules` and sit at the project root.

## 4. Verify the Setup

Open Cascade (`Ctrl+L` or the Cascade panel) and ask:

```
What are the Salesforce development rules for this project?
```

Cascade should list the rules from `.windsurfrules`. If it doesn't, close and reopen the folder.

## 5. Multi-File Context with sfdx-project.json

Cascade is aware of multiple open files simultaneously. Keep these open alongside your rules for Salesforce work:

| Open File | Why It Helps |
|---|---|
| `sfdx-project.json` | Cascade understands your package directories and API version |
| `manifest/package.xml` | Cascade knows which metadata types are in scope |
| Related `.field-meta.xml` | Cascade uses actual field names and picklist values |
| Existing trigger handler | Cascade extends the right class instead of creating a new trigger |

## 6. Tips for Using Cascade on Salesforce Projects

- Cascade can write multiple files in one pass. Start with a description of the full feature (trigger, handler, service, test) rather than asking for one file at a time.
- Before running any deploy command Cascade suggests, verify it targets the right org alias. Cascade doesn't know which org is your sandbox vs production.
- After Cascade writes Apex, ask it: "Score this Apex class against multi-tenant governor limit rules before we deploy."
