# Using SF Skills with Claude Code

## Setup

Copy the skills you want into your project's `.claude/skills/` directory:

```bash
# copy all core skills
cp -r path/to/SF-AI-Knowledgehub/skills/sf-apex .claude/skills/
cp -r path/to/SF-AI-Knowledgehub/skills/sf-lwc .claude/skills/
# repeat for each skill you want
```

Or copy all at once:
```bash
for skill in sf-apex sf-lwc sf-flow sf-security sf-test sf-soql sf-debug sf-deploy sf-permission sf-integration; do
  cp -r path/to/SF-AI-Knowledgehub/skills/$skill .claude/skills/
done
```

## Invoke a skill

Type the skill name with a leading slash in Claude Code:

```
/sf-apex
/sf-lwc
/sf-flow
```

Claude loads the skill into context for the session.

## When to invoke which skill

| You're about to... | Invoke |
|-------------------|--------|
| Write a new Apex class or trigger | `/sf-apex` |
| Write a new LWC component | `/sf-lwc` |
| Build or modify a Flow | `/sf-flow` |
| Do a pre-deploy security review | `/sf-security` |
| Write a test class | `/sf-test` |
| Write or optimise a SOQL query | `/sf-soql` |
| Debug an error or unexpected behaviour | `/sf-debug` |
| Plan or execute a deployment | `/sf-deploy` |
| Set up Permission Sets or sharing | `/sf-permission` |
| Build a callout or inbound REST API | `/sf-integration` |

## Skill invocation rules

- Invoke only when you need it -- each skill loads into context and stays there
- Don't invoke multiple skills at once unless you're working across both areas
- For a new feature spanning Apex + LWC + deploy: invoke in order as you work each layer
