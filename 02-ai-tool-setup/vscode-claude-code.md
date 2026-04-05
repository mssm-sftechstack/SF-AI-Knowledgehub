# Claude Code in VS Code vs Terminal

Claude Code runs in two places: the VS Code extension and the terminal CLI. They use the same model and the same `CLAUDE.md` context file, but the experience is different. This guide covers when to use each and how to get them working together.

---

## The Two Modes

| | VS Code Extension | Terminal CLI |
|---|---|---|
| How you open it | Side panel or `Ctrl+Shift+P` -> Claude Code | `claude` in your project folder |
| Context it reads | `CLAUDE.md` in your workspace root | `CLAUDE.md` in your current directory |
| File awareness | Sees your open files and current selection | Sees whatever you paste or reference |
| Best for | Quick edits, explaining code, reviewing a file you have open | Multi-file tasks, running commands, deploy pipelines |
| Inline diffs | Yes -- applies changes directly to open files | Yes -- same behaviour |
| Runs shell commands | Yes, with approval | Yes, with approval |
| Skills (`/sf-apex` etc.) | Yes | Yes |

Both modes share the same session history within a conversation, but not across restarts.

---

## Install the VS Code Extension

1. Open VS Code
2. Extensions sidebar (`Ctrl+Shift+X`)
3. Search "Claude Code"
4. Install the Anthropic extension
5. Sign in when prompted (same account as `claude auth login`)

You'll see a Claude Code icon appear in the Activity Bar on the left.

---

## Install the Terminal CLI

```bash
npm install -g @anthropic-ai/claude-code
```

Then from your SF project folder:

```bash
cd your-sf-project
claude
```

If you haven't authenticated yet:

```bash
claude auth login
```

---

## How the Extension Works

### Opening it

Three ways:
- Click the Claude Code icon in the Activity Bar
- `Ctrl+Shift+P` -> type "Claude Code: Open"
- Keyboard shortcut (you can set this in Keybindings)

### Sending your current file

When you have a file open, Claude Code can see it. You don't need to paste content -- just ask "what does this class do?" and it reads the open file.

To reference a specific selection: highlight the code first, then open Claude Code. The selection appears as context automatically.

### Applying edits

When Claude suggests a code change, you'll see a diff view. Accept or reject each change individually. You can also ask it to redo a change if the first attempt wasn't right.

### Running commands

The extension can run shell commands (like `sf project deploy` or `git status`) with your approval. Each command shows before it runs. You can deny individual commands.

---

## How the Terminal CLI Works

### Starting a session

```bash
cd your-sf-project
claude
```

The `>` prompt is your entry point. Type naturally.

### Resuming a session

```bash
claude -c
```

This continues the most recent session with full context preserved -- useful when you close and reopen the terminal.

### Referencing files

You can reference files directly:

```bash
> read force-app/main/default/classes/AccountService.cls and review for FLS issues
```

Or ask it to run a glob:

```bash
> find all Apex classes that do DML without CRUD checks
```

### Running deploy commands

The terminal CLI is better for anything involving shell commands in sequence:

```bash
> validate the deploy, then if it passes run the tests, then show me coverage
```

It'll walk through each step and ask approval before running.

---

## Which One to Use

**Use the VS Code extension when:**
- You're actively editing a file and want feedback on what's on screen
- You want to explain, document, or refactor a specific method
- You want inline diffs applied directly in the editor
- You're doing quick one-off questions about open code

**Use the terminal CLI when:**
- You're running the `/sf-build` pipeline or multi-step deploy workflow
- You need it to orchestrate across multiple files
- You're running shell commands (deploy, test runs, git)
- You want to resume a long session with `claude -c`
- You're using skills like `/sf-apex` or `/sf-security` that trigger agent pipelines

**You can use both at the same time.** The extension is good for the code you're editing right now; the terminal handles the surrounding workflow.

---

## CLAUDE.md Works the Same in Both

Both modes read `CLAUDE.md` from your project root. Copy the template from [templates/CLAUDE.md](../templates/CLAUDE.md) into your project root once and both the extension and terminal will pick it up.

```bash
cp templates/CLAUDE.md /path/to/your/sf-project/CLAUDE.md
```

The `CLAUDE.md` sets your org alias, API version, skills registry, hard rules, and deploy patterns. Claude reads it at the start of every session in both modes.

---

## Skills in the Extension

Skills work the same in the extension as in the terminal. If you've copied skills into `.claude/skills/` in your project:

```
your-sf-project/
  .claude/
    skills/
      sf-apex/
        SKILL.md
```

Then `/sf-apex` works in both the extension chat and the terminal prompt.

To use a skill from the [SF-AI-Knowledgehub skills folder](../skills/), copy the `SKILL.md` file into `.claude/skills/<skill-name>/SKILL.md` in your project.

---

## Keyboard Shortcuts (VS Code Extension)

| Action | Default Shortcut |
|---|---|
| Open Claude Code panel | `Ctrl+Shift+P` -> "Claude Code: Open" |
| Send selection as context | Highlight code, then open panel |
| Accept all suggested changes | Button in diff view |
| Reject all suggested changes | Button in diff view |

You can customise shortcuts in VS Code: `Ctrl+K Ctrl+S` -> search "Claude Code".

---

## Troubleshooting

**Extension not seeing my CLAUDE.md**
Make sure `CLAUDE.md` is in the workspace root -- the same folder you opened in VS Code (`File -> Open Folder`). If you opened a subfolder, move up to the project root.

**Terminal CLI picking up wrong directory**
Always run `claude` from your project root. The session context is tied to the directory you're in when you start.

**Authentication issues in extension**
Sign out and back in: `Ctrl+Shift+P` -> "Claude Code: Sign Out", then sign in again. The extension and CLI share credentials, so signing in via `claude auth login` in the terminal also fixes extension auth.

**Skills not loading**
Check that your skill is at `.claude/skills/<name>/SKILL.md` (not `.claude/skills/<name>/<name>.md`). The frontmatter `name:` field must match the folder name.
