# Using SF Skills with Any AI Tool

These skills are plain markdown. They work wherever you can paste text.

## For chat-based AI tools

Paste the skill's README.md content at the start of your conversation:

```
I'm working on a Salesforce project. Apply these rules throughout our session:

[paste content of skills/sf-apex/README.md]

Now help me build [your feature].
```

## For tools with system prompt support

Paste the skill content as your system prompt or custom instructions. Most tools support this:
- ChatGPT: Custom Instructions
- Gemini: System prompt in API / Gems
- Claude.ai: Custom Instructions in Projects
- Copilot Studio: System topic instructions

## For any IDE extension

If your AI extension has a "context" or "custom instructions" field, paste the relevant skill there. The skill is just text -- it works anywhere text context is supported.

## Minimal version

If context space is tight, use just the rules sections (skip the code examples). The rules headings and bullet points are the most important part.
