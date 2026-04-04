# Claude Code Skills Guide for Salesforce

**Skills are reusable instruction sets that Claude Code loads on demand. Load only what you need for the current task.**

---

## What a Skill Is

A skill is a `.md` file at `.claude/skills/<skill-name>/SKILL.md`. Type `/skill-name` in a Claude Code session and Claude loads those instructions into its context for that session.

This keeps your main `CLAUDE.md` lean. Instead of loading every possible Salesforce rule upfront, you load the Apex skill when writing Apex, the LWC skill when building components, and the security skill before every deploy.

---

## Core Salesforce Skills

| Skill | When to Invoke | What It Enforces |
|---|---|---|
| `/sf-build` | Any new feature, class, LWC, or trigger | Full Design → Code → Review → Safeguard → DevOps pipeline |
| `/sf-apex` | Apex class development | Bulkification, FLS, sharing, async patterns, 150-point scoring |
| `/sf-lwc` | LWC component development | LDS first, accessibility, security, 165-point scoring |
| `/sf-flow` | Flow development | Bypass logic, fault paths, Transform over Loop, 110-point scoring |
| `/sf-security` | Pre-deployment security review | SOQL injection, XSS, FLS, CRUD, sharing gaps |
| `/sf-test` | Apex test class generation | 90%+ coverage, 200-record bulk test, runAs, Mixed DML |
| `/sf-deploy` | Deployment planning | Dry-run first, deployment order, phase validation |
| `/sf-soql` | SOQL authoring | Indexes, selectivity, anti-patterns, query scoring |
| `/sf-debug` | Error diagnosis | Log analysis, error root cause table, Limits tracking |
| `/sf-metadata` | Metadata authoring | package.xml, sfdx-project.json, metadata type reference |
| `/sf-ai-agentforce` | Agentforce development | Topics, Actions, InvocableMethod patterns |
| `/sf-permission` | Permission model | Permission Sets, FLS, sharing model, hierarchy visualisation |

---

## How to Set Up Skills

Copy the skills directory from this repo into your project:

```bash
cp -r SF-AI-Knowledgehub/templates/claude-skills/ .claude/skills/
```

Your project structure should look like:

```
your-sf-project/
  CLAUDE.md
  .claude/
    settings.json
    skills/
      sf-build/
        SKILL.md
      sf-apex/
        SKILL.md
      sf-lwc/
        SKILL.md
      sf-security/
        SKILL.md
      ...
  force-app/
  manifest/
```

---

## Skill Invocation Rules

- Invoke skills only when needed. Each one loads into context and stays there for the session.
- `/sf-build` is the entry point for all new features. It calls the other skills internally. Don't invoke `/sf-apex` or `/sf-lwc` manually when building a new feature.
- `/sf-security` always runs before any deploy. Never skip it.
- Don't invoke the same skill twice in one session.

---

## The `/sf-build` Pipeline

`/sf-build` runs a full multi-agent pipeline:

```
/sf-build
  → Design Agent       (architecture, data model, API design)
  → User Approval      (you review the plan before any code is written)
  → Code Agent         (writes Apex, LWC, metadata, test classes)
  → Code Review Agent  (style, patterns, architecture)
  → Safeguard Agent    (security — runs /sf-security automatically)
  → DevOps Agent       (deployment plan, package.xml, order of operations)
  → FEATURE_REVIEW     (consolidated report — you approve before any deploy)
```

Don't start a new feature by asking Claude to "write an Apex class" directly. Always start with `/sf-build` to get the design-first workflow.

---

## Skill Scoring

The Apex, LWC, and Flow skills use scoring rubrics before marking a component ready to ship:

| Component | Pass Threshold | Total Points |
|---|---|---|
| Apex Class | 120+ | 150 |
| LWC Component | 132+ | 165 |
| Flow | 88+ | 110 |

If a component scores below the threshold, Claude lists the deductions and fixes them before presenting the code for review.
