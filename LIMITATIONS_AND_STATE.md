# Limitations and Current State

What works. What doesn't. What's still experimental. Read this before using the project.

---

## Implementation Status

### ✅ Fully Implemented

**Context Templates** (production-ready within your own org)
- `CLAUDE.md` for Claude Code — validated with real projects
- `.cursorrules` for Cursor — translated from CLAUDE.md
- `copilot-instructions.md` for GitHub Copilot — API version tested
- `.windsurfrules` for Windsurf — syntax-verified
- `AGENTS.md` generic template for any AI tool
- All templates include org-specific placeholders for customization

**Core Guardrails** (10 documented patterns)
- Tested against 10+ production Salesforce projects
- Each guardrail comes with rationale and a real-world failure scenario
- Covers: bulkification, SOQL, security model, deployment order, testing

**Reference Materials** (lookup-ready)
- Governor limits reference (complete)
- Deployment order (all 16 phases, verified)
- Common error patterns (50+ documented)
- Metadata type reference (complete)

**Architecture Patterns** (documented, not auto-generated)
- Layered Apex architecture (selector → service → trigger handler)
- Async decision tree (Queueable vs Schedulable vs Platform Events)
- SOQL performance fundamentals (indexes, selectivity, limits)
- Order of execution reference (triggers, flows, platform events)
- Multi-tenant risk checklist

**Agentforce Patterns** (learned from Salesforce docs + experience)
- Invocable Apex patterns (annotations, request/response classes)
- System prompt design (scope, escalation, PII handling)
- Trust Layer reference (what it does and doesn't mask)
- Topic + Action architecture

---

### ⚠️ Experimental (Works but Still Evolving)

**Session Memory Across AI Tools**
- Works in one session within one project. Doesn't work across projects.
- What actually works: CLAUDE.md hooks load context for every prompt in that session.
- What doesn't: If you start a new project, the AI forgets. You reload the template from scratch.
- What we're learning: Memory needs discipline. SESSION_STATE.md and HANDOFF.md create structure, but without commitment to updating them, memory gets lost.
- Use this for now: Templates plus session files. Don't expect the AI to automatically carry lessons between projects.

**AI Tool Compatibility**
- Claude Code: Tested and reliable with CLAUDE.md.
- Cursor: The syntax is correct. But it doesn't behave exactly like Claude Code in edge cases (rule priority, multi-file context).
- GitHub Copilot: Works for inline code, but not as good for architecture questions.
- Windsurf: Syntax is right. But almost no real-world testing.
- Start with Claude Code. Other tools work maybe 70-80% as well. You'll need to adjust as you go.

**CI/CD GitHub Actions**
- JWT auth pattern is documented. Example workflows included. But not tested end-to-end.
- Documented: PR validation, deploy-on-merge, secret management.
- Not tested: Partial deployments, rollbacks, multi-org pipelines.
- Use this as a starting point. Your CI/CD needs org-specific customization.

**Agentforce Patterns**
- These are based on Salesforce v1.0. Agentforce moves fast.
- Not covered yet: Agent memory and persistence (new in later versions).
- Not covered yet: Complex multi-topic routing.
- Not covered yet: Advanced persona techniques.
- Use the Trust Layer and invocable Apex patterns as your base. Then watch Salesforce release notes for new Agent features.

---

### ❌ Not Implemented

**Production Deployment Automation**
- Why: Too org-specific. Deployment strategy varies by: release cadence, team size, approval workflows, rollback procedures.
- What to use instead: The deployment order reference + your own CI/CD pipeline (see the GitHub Actions starting point).

**Platform Cache Partition Management**
- Why: Partitions can't be created via metadata API. They must be created manually in Setup UI.
- Current workaround: Document your partition names and TTLs in a manual setup checklist.
- Reference: [07-devops/deployment-strategy.md](07-devops/deployment-strategy.md#cross-phase-traps-that-cause-failures)

**Scratch Org Seeding**
- Why: Seed data is org-specific (accounts, contacts, permissions, custom settings).
- What's documented: How to set up scratch orgs. Not: What to put in them.
- Recommendation: Build your own seed scripts using the scratch org quickstart as a template.

**LWC Component Library**
- Why: Components are highly specific to feature sets. General patterns exist, but a reusable library would be misleading.
- What's documented: LWC patterns (wire adapters, state management, testing).
- What's not: Pre-built reusable components.

**Flow Orchestrator Patterns**
- Why: Flow Orchestrator is newer and use cases are still emerging.
- What's documented: Basic orchestration concepts, subflow patterns.
- What's not: Multi-phase orchestration, error recovery, state management across phases.

**Multi-Org Deployment Strategies**
- Why: Multi-org governance is highly specific to org topology, approval workflows, and compliance requirements.
- What's documented: Single-target deployments.
- What's not: Change sets, org-to-org migrations, governance models.

**Change Data Capture (CDC) Integration Patterns**
- Why: CDC is powerful but org-specific. Too many variables.
- What's documented: Async decision tree (Queueable vs Schedulable vs CDC).
- What's not: Real-world CDC consumer patterns, event-driven architecture.

---

## Assumptions Made

These assumptions hold the project together. Violate them, and you'll need to adapt:

| Assumption | Why It Matters | If It's Wrong |
|---|---|---|
| **Modern SF CLI (`sf`, not `sfdx`)** | Docs use current syntax. Old commands work differently. | Use `sfdx`. Some flags might differ. |
| **Named scratch orgs** | Setup guides assume you use org aliases locally. | You'll need to adjust auth setup. |
| **Permission Sets, not Profiles** | Profiles are legacy. Docs focus on Permission Sets. | Adapt the permission section for Profiles. |
| **API v62+ (Spring 24+)** | Hard rules reference v62 limits. Older versions differ. | Check your API version's limits. |
| **VS Code as IDE** | Setup guides assume VS Code. | Adjust for your IDE and extensions. |
| **Salesforce CLI installed globally** | Guides assume `sf` is in your PATH. | Install globally or use a container. |
| **English-language prompts** | AI instructions are in English. | Non-English templates don't exist yet. |
| **Single-team, Dev to QA to Prod** | Deployment examples assume this flow. | Complex approvals need custom pipelines. |

---

## Known Gaps

Specific things not yet documented (contributions welcome):

**Salesforce Features Not Covered**
- Managed Packages (ISV patterns)
- Experience Cloud (guest users, sharing models for external users)
- Communities and Service Cloud (case routing, omnichannel)
- Financial Services Cloud, Health Cloud, etc. (industry-specific)
- Einstein Analytics (data modeling for analytics)
- Salesforce Maps, Tableau integration
- Field Service Lightning (scheduling, resource management)

**AI Tool Features Not Tested**
- Claude Code workspaces (multi-file chat context)
- Cursor agents for code generation
- GitHub Copilot Workspace mode
- Windsurf's cascade agent for large refactors

**DevOps Scenarios Not Documented**
- Blue-green deployments
- Feature flags for gradual rollouts
- Dependency-driven deployment order (when object A depends on B, which depends on C)
- Rollback strategies for data migrations
- Production data masking for dev/test orgs

**Testing Scenarios Under-Documented**
- Contract testing for APIs
- Visual regression testing for LWC
- Load testing (finding governor limit bottlenecks)
- Security scanning in CI/CD

**Salesforce Metadata Oddities**
- Custom Metadata serialization (CMDT isn't always deployable)
- Dependent metadata (e.g., some field metadata can't deploy without the object)
- Org-wide defaults (OWD) and how they interact with sharing rules

---

## Limitations by Use Case

### When This Won't Help

**Advanced Salesforce Features**
Industries Cloud, FSC, Health Cloud. These aren't covered. You need specialized patterns.

**Legacy Orgs**
API v58 or earlier. Profiles only (no Permission Sets). Custom CLI setups. Templates need heavy adaptation.

**ISV / Managed Packages**
The multi-tenancy model here is for subscriber orgs, not ISVs. Different patterns needed.

**Complex Multi-Org Governance**
10+ orgs with approval workflows spanning them. The single-org assumption breaks down.

**Non-English Teams**
AI instructions are English-only. No translations exist.

**Non-VS Code IDEs**
IntelliJ, Sublime, Vim. Setup guides won't match. You'll adapt them.

**Air-Gapped / Offline**
Can't use AI tools for code generation without internet.

---

## What May Not Work Reliably

**AI Tool Consistency Across Versions**
- Templates assume Claude Code v1.0+, Cursor v0.36+, Copilot v1.140+, Windsurf v1.0+.
- Newer versions might change how contexts load, rule priority, context window handling.
- Fix: Pin tool versions in your setup guide, or refresh templates every quarter.

**Context Templates Get Stale**
- Salesforce updates quarterly. New features, new limits, new metadata types.
- Templates include hard rules (stable), but also links and API version assumptions (can change).
- Stay on top of release notes. Update template links when each release drops.

**Bulkification Patterns May Fail at Edge Cases**
- Guardrails assume 200-record bulk tests, one trigger per object, basic async.
- Reality: Some orgs have 10,000+ records per transaction, complex async chains, or before-insert dependencies.
- These patterns handle 80% of cases. Document your edge cases in team docs.

**Memory System Breaks on Team Scaling**
- Works for: single developer, 1-2 projects.
- Breaks at: 5+ developers, 10+ projects, team handoffs.
- Why: Session memory is person-scoped. Team-scoped memory needs infrastructure (shared docs, wikis, etc.).
- Mitigation: Use this as a starting point. Graduate to team wikis (Notion, Confluence) at scale.

**AI Tool Performance Degrades with Large Org Schema**
- Templates work for: orgs with <500 custom fields, <50 custom objects.
- May struggle with: massive orgs (lots of metadata = bigger context payload).
- Mitigation: Filter CLAUDE.md to only relevant objects/fields for that feature.

---

## Test Coverage by Component

How thoroughly each section has been tested:

| Component | Tested In | Real Orgs | Edge Cases | Recommendation |
|---|---|---|---|---|
| Core guardrails (10 rules) | 10+ projects | ✅ Yes | ✅ Covered | Use as-is |
| Claude Code template | 5+ projects | ✅ Yes | ✅ Covered | Use as-is |
| Cursor template | 2 projects | ⚠️ Limited | ⚠️ Some | Tweak for your org |
| Copilot template | 1 project | ⚠️ Limited | ❌ No | Expect friction |
| Windsurf template | Doc review only | ❌ No | ❌ No | High risk, adapt first |
| Deployment order | 10+ deploys | ✅ Yes | ✅ Covered | Use as-is |
| Layered Apex pattern | 15+ classes | ✅ Yes | ✅ Covered | Use as-is |
| Async decision tree | 5+ async patterns | ⚠️ Limited | ⚠️ Some | Review for your case |
| Permission Set patterns | 8+ PS designs | ✅ Yes | ⚠️ Some | Use, watch for edge cases |
| SOQL performance guide | 20+ queries | ✅ Yes | ✅ Covered | Use as-is |
| Agentforce patterns | 2 agents | ⚠️ Limited | ❌ No | Test before prod |
| CI/CD GitHub Actions | 1 org | ⚠️ Limited | ❌ No | Customize heavily |

---

## How This Will Change

**Stability Guarantees**
- Templates (CLAUDE.md, .cursorrules, etc.): Backward-compatible updates only. Breaking changes get major version bumps.
- Guardrails (10 core rules): Stable. Only updated if Salesforce changes governor limits.
- Architecture patterns: Stable, with new patterns added (not removed).
- Reference materials: Updated quarterly with Salesforce release notes.

**What May Break**
- Links to Salesforce docs (Salesforce moves things around).
- AI tool template syntax (tools update their formats).
- Example YAML for CI/CD (GitHub Actions syntax evolves).

**Contribution Impact**
- When you contribute patterns from your org, they're tested only in your context.
- Other teams may need to adapt them.
- The more orgs test a pattern, the more we understand its limits.

---

## Contributing Improvements

**If you find something that doesn't work:**
1. Document what you expected, what happened, and your org context (API version, features you use).
2. Share in [CONTRIBUTING.md](CONTRIBUTING.md) what you changed to fix it.
3. We'll integrate it and note the edge case.

**If you want to expand coverage:**
- Add patterns you've used in 2+ projects (untested patterns waste time).
- Include rationale, failure modes, and the orgs you tested it in.
- Cross-check against official Salesforce docs before submitting.

---

## Bottom Line

**Use this for:**
- Learning how to work with AI tools in Salesforce
- Enforcing best practices across your team's AI use
- Stopping repeated debugging of the same mistakes
- Building your own deployment playbook

**Don't use this instead of:**
- Architecture reviews
- Security audits
- Your org's governance
- A senior engineer thinking through edge cases
- Official Salesforce documentation

Use it as a foundation. Make it your own. Share back what works.
