# Claude Code Setup for Salesforce

**Claude Code is Anthropic's CLI for Claude. It reads `CLAUDE.md` as persistent project context at the start of every session.**

---

## 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Then launch it from your project directory:

```bash
cd your-sf-project
claude
```

## 2. Authenticate

```bash
claude auth login
```

This opens a browser for OAuth. Your credentials are stored locally once you're done.

## 3. Add Your CLAUDE.md

Copy the template into your project root:

```bash
cp path/to/SF-AI-Knowledgehub/templates/CLAUDE.md ./CLAUDE.md
```

Open it and fill in the two placeholders marked `# FILL IN`:

- `<your-org-alias>` — the alias you use with `sf org login`
- `<your-instance-url>` — your org's My Domain URL (e.g. `https://mycompany.my.salesforce.com`)

## 4. What CLAUDE.md Does

Claude Code reads `CLAUDE.md` automatically at session start. It sets:

- **Org context** — alias, API version, deploy and retrieve commands
- **Hard rules** — non-negotiable rules Claude applies to every file it writes
- **Deployment order** — the sequence Claude follows when planning a multi-file deploy

Claude won't generate code that violates these rules. If you ask it to put SOQL in a loop, it'll refuse and explain why.

## 5. Add Skills (Optional but Recommended)

Skills are reusable instruction sets for specific tasks — Apex, LWC, Flow, security review, etc. Claude loads them on demand, keeping your main context lean.

Copy the pre-built skills into your project:

```bash
cp -r path/to/SF-AI-Knowledgehub/templates/claude-skills/ .claude/skills/
```

See the full guide: [skills-guide.md](skills-guide.md)

## 6. Add Deployment Hooks (Optional)

Hooks block dangerous actions automatically. Add them to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/deploy-gate.js"
          }
        ]
      }
    ]
  }
}
```

A deploy gate hook checks for your explicit confirmation before any `sf project deploy` command runs.

## 7. Verify the Setup

Ask Claude Code:

```
What are the hard rules for this project?
```

It should recite the rules from your `CLAUDE.md`. If it doesn't, check that `CLAUDE.md` is in the project root — the same folder where you ran `claude`.

---

## File Locations Summary

| File | Purpose |
|---|---|
| `CLAUDE.md` | Project rules, org context, deployment order |
| `.claude/skills/` | Skill files for Apex, LWC, Flow, Security, etc. |
| `.claude/settings.json` | Hooks, tool permissions, model preferences |
