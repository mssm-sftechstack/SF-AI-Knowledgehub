# Using SF Skills with Cursor

Cursor reads `.cursorrules` from your project root. You can load skill content directly into that file.

## Option A: Paste the full skill

Open the skill's `README.md`, copy all content, and paste it into your `.cursorrules` under a clear header:

```
## SF Apex Skill

[paste content of skills/sf-apex/README.md here]
```

## Option B: Load multiple skills

Chain multiple skills in `.cursorrules`:

```
## SF Skills loaded for this project

### Apex Rules
[paste sf-apex/README.md]

### LWC Rules
[paste sf-lwc/README.md]

### Security Rules
[paste sf-security/README.md]
```

## Tip: keep .cursorrules lean

Loading too many skills at once bloats your context window and degrades suggestion quality. Load the 2-3 skills most relevant to what you're building right now.

## Verify it's working

Open Cursor chat and ask: "What's the correct Apex architecture pattern for this project?" It should describe Trigger -> Handler -> Service -> Selector. If it does, the skill is active.
