# Using SF Skills with Windsurf

Windsurf's Cascade agent reads `.windsurfrules` from your project root.

## Setup

Paste skill content into `.windsurfrules`:

```
## SF Apex Skill Rules

[paste content of skills/sf-apex/README.md]
```

## Windsurf-specific tip

Cascade is multi-file aware. Open your sfdx-project.json alongside the skill content -- Cascade will use both to understand the project structure and apply the right rules.

## Loading multiple skills

Same approach as Cursor -- paste under clear headers. Keep to 2-3 skills per session.

## Verify

Ask Cascade: "What async pattern should I use for a callout from trigger context?" It should say Queueable with Database.AllowsCallouts, not @future.
