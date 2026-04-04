# Cursor Setup for Salesforce

**Cursor reads `.cursorrules` from your project root as persistent instructions for every chat and Cmd+K interaction.**

---

## 1. Install Cursor

Download from [cursor.sh](https://cursor.sh) and install. Cursor is a VS Code fork. If you already use VS Code, your extensions and settings transfer over.

## 2. Open Your Salesforce Project

```
File → Open Folder → select your SFDX project root
```

The project root is the folder containing `sfdx-project.json`.

## 3. Add the Rules File

Copy the template to your project root:

```bash
cp SF-AI-Knowledgehub/templates/.cursorrules ./.cursorrules
```

The file must be named exactly `.cursorrules` and sit at the root next to `sfdx-project.json`.

## 4. Verify the Setup

Open Cursor's chat panel (`Ctrl+L` or `Cmd+L`) and ask:

```
What are the Salesforce guardrails for this project?
```

Cursor should list the rules from `.cursorrules`. If it doesn't, close and reopen the project folder.

## 5. Keep Context Clean with `.cursorignore`

Add a `.cursorignore` file at your project root to stop Cursor from indexing irrelevant files. This keeps its context window focused on your Salesforce code:

```
# .cursorignore
node_modules/
.sfdx/
.sf/
*.log
*.xml.bak
coverage/
```

---

## How Cursor Uses the Rules

| Interaction | Rules Applied |
|---|---|
| Chat (`Ctrl+L`) | Yes, full `.cursorrules` context |
| Cmd+K inline edit | Yes, full `.cursorrules` context |
| Autocomplete (Tab) | Partial. Model may apply some rules. |
| Composer (multi-file) | Yes, full `.cursorrules` context |

Chat and Composer are the most reliable ways to enforce rules. The full context file is always loaded there.

---

## Tips for Salesforce Development in Cursor

- When writing a new Apex class, open the related test class in a split pane. Cursor uses open files as additional context.
- For SOQL queries, paste the object's field list from the org metadata into the chat alongside your question.
- Use Composer for multi-file features (trigger + handler + service + test). It can write all files in one pass with the rules applied.
