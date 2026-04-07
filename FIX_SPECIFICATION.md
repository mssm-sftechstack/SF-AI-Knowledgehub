# CTA Quality Fix Specification

**Completed**: ORDER_OF_EXECUTION.md (1/15)

**Pattern Established**:
- Remove meta-commentary opening ("This document explains...")
- Add mermaid sequenceDiagram or flowchart at top
- Expand each concept to show implications for Apex, LWC, Flow, Metadata
- Strip AI filler ("It's important to note...", "This ensures that...")
- Remove em dashes (use parentheses or separate sentences)
- Add "From X perspective:" subsections (From Apex:, From LWC:, From Flow:, etc.)

---

## Remaining 14 Files: Fix Checklist

### Core Concepts (9 files)

#### FLOW_BEST_PRACTICES.md
- **Current issue**: Apex-centric (Use Flow when... Use Apex when...). Reads like Apex is the default.
- **Fix**: Reframe as "Flow is the tool. Here's when it applies." Remove "Use Apex" comparison table.
- **Add**: 
  - Mermaid flowchart: decision tree (simple logic? → Flow; callout needed? → Apex; user input? → Screen Flow)
  - "Flow Limits & Constraints" section showing: max 25MB data, no direct callout, recursion limits
  - "From LWC perspective: How does Flow visibility affect sync/async?"
  - "Subflow recursion prevention" with explicit guard logic (not just mention it)
- **Remove**: "Don't Use Process Builder" (outdated, unnecessary)

#### INTEGRATION.md
- **Current issue**: 90% Apex callouts (Apex HTTP, Named Credentials, retry). Missing Flow/LWC patterns.
- **Fix**:
  - Add mermaid sequenceDiagram: request → Named Credential → callout → response
  - Split sections: "Apex Callouts" + "Flow Callouts (via Apex action)" + "LWC Callouts (via Apex controller)"
  - "Named Credentials" section: explain both REST Named Cred + OAuth Named Cred, when each used
  - "Retry Logic": Queueable pattern + Flow error handling pattern (fault path)
  - Remove marketing ("Named Credentials are the best practice..." → "Use Named Credentials to centralize auth.")
- **Add diagrams**:
  - Callout flow for Apex, Flow, LWC (each different)
  - Retry logic decision tree

#### SECURITY.md
- **Current issue**: Mostly Apex (SOQL injection, FLS, CRUD). Missing Flow/LWC/Metadata security.
- **Fix**:
  - Expand: "CRUD/FLS enforcement across all components"
  - Add sections: "LWC Security" (sanitization, XSS), "Flow Security" (sharing model), "Metadata Security" (permission model)
  - Add mermaid flowchart: user requests data → FLS check → sharing check → CRUD check → return result
  - "Hardcoded IDs" section: show Apex + Flow + LWC examples of wrong/right
  - Remove vague statements ("important for compliance")
- **Add diagrams**: FLS + CRUD + Sharing flow

#### LWC_BEST_PRACTICES.md
- **Current issue**: This is fine conceptually, but needs diagrams and explicit non-Apex cross-references.
- **Fix**:
  - Add mermaid diagram: component lifecycle (create → mount → update → destroy)
  - "Wire Adapter Patterns": show vs. imperative vs. refreshData()
  - "State Management": reference Apex context (triggers update LWC? Flow updates?)
  - "Error Handling": show both local error + server error paths
  - Add "From Order of Execution perspective: When does LWC see trigger/flow changes?"

#### TESTING_STRATEGY.md
- **Current issue**: 90% unit testing Apex (test classes, assertions). Missing Flow/LWC testing, integration test patterns.
- **Fix**:
  - Add mermaid flowchart: unit vs integration vs bulk decision tree
  - Section: "Apex Unit Testing" (current content, but clarify System.runAs requirement)
  - New section: "Flow Testing" (e.g., how to test record-triggered flows in test context?)
  - New section: "LWC Testing" (Jest patterns, wire adapter mocks) — defer to jest skill
  - New section: "Bulk Testing" (200+ records, measure performance)
  - New section: "Integration Testing" (test trigger + flow together)
  - Add explicit 90%+ coverage requirement with example failing test
- **Add diagrams**: Test pyramid (unit → integration → e2e)

#### PERMISSIONS.md
- **Current issue**: Mostly PermissionSet + field permissions (Apex-centric). Missing Flows, LWC, custom metadata.
- **Fix**:
  - Reframe: "Permission Model" not just "PermissionSets"
  - Add sections: "Permission Set" + "Custom Permissions" + "Org-Wide Defaults" + "Sharing Rules"
  - "Field Permissions" section: explain fieldPermissions in PS (required for all custom fields or FLS violation)
  - New: "Flow Security" (which users can see/run which flows)
  - New: "LWC Security" (LWC visibility vs Apex service permissions)
  - Add "Testing" section: System.runAs pattern for permission testing
- **Add diagrams**: Permission model hierarchy (OWD → Role Hierarchy → Sharing Rules → Record Owner)

#### AI_PITFALLS.md
- **Current issue**: Called "AI Pitfalls" but really just 12 common mistakes. Most are Apex.
- **Fix**: Rename to "Common Pitfalls" (or keep "AI" but clarify it means "when AI helps you code").
  - Current pitfalls (bulkification, SOQL in loops, etc.): Keep, but add LWC/Flow/Integration examples too
  - Add pitfalls missing: 
    - "Assuming all users see all records" (sharing model)
    - "Hardcoded IDs" (examples in Flow, Apex, LWC)
    - "Order of execution surprises" (cross-reference ORDER_OF_EXECUTION.md)
    - "Flow infinite recursion" (with guard pattern)
    - "LWC not waiting for async Apex" (user sees stale data)
  - Pitfall format: Wrong → Right → Why → Where it bites you
- **Add diagrams**: One diagram per 3 pitfalls showing the error flow

#### MULTITENANT.md
- **Current issue**: Governor limits focus (mostly Apex). Missing Flow/LWC pod architecture impacts.
- **Fix**:
  - Add "Pod Architecture" section: which components run on which pods, why it matters
  - "Governor Limits by Component": table showing Apex limits + Flow limits + LWC limits
  - "Resource Contention": explain how one bad trigger/flow can slow down ALL users on same pod
  - "Scaling strategies": batch processing, queueable chaining, flow limits
  - Add mermaid diagram: multi-tenant architecture (orgs → pods → servers)
- **Add diagrams**: Multi-tenant pod layout, resource contention flow

### Reference Guides (5 files)

#### reference/sf-cli-cheatsheet.md
- **Current issue**: Just a list. No marketing, but verify all commands are current.
- **Fix**:
  - Verify all `sf` CLI commands are valid (not deprecated `sfdx`)
  - Add error codes section (common errors + fixes)
  - Organize by task (deploy, retrieve, test, query, etc.)
  - Verify URLs and version numbers
  - No marketing needed; just facts.

#### reference/soql-anti-patterns.md
- **Current issue**: Lists anti-patterns but no diagrams. Lacks practical "how to fix" guidance.
- **Fix**:
  - Add mermaid diagram: query execution path (parse → index lookup → scan → filter → return)
  - Each anti-pattern: Wrong → How to fix → Performance impact
  - Add "Index usage" section: how Salesforce chooses indexes
  - Add "Query plan" section: how to read SOQL query plan
  - Verify all examples are correct Apex syntax

#### reference/batch-apex-patterns.md
- **Current issue**: Batch vs Queueable vs Scheduled decision tree exists, but needs diagrams.
- **Fix**:
  - Add mermaid flowchart: "When to use: Batch vs Scheduled vs Queueable"
  - "Batch Pattern": start/execute/finish with error handling
  - "Queueable Pattern": single execute, chaining, callout
  - "Scheduled Pattern": cron expression, callout via Queueable
  - Verify all syntax is valid Apex 2025+
  - Add "Testing" section: how to test scheduled/batch in test context

#### reference/lwc-security.md
- **Current issue**: Existence is good, but make sure it's comprehensive.
- **Fix**:
  - Add mermaid diagram: input → sanitization → DOM injection (safe vs unsafe)
  - "XSS Prevention": HTMLElement vs innerHTML/textContent
  - "Sanitization": dompurify, LightningElement, safe patterns
  - "DOMPurify library": when to use, setup
  - Verify examples match current LWC best practices
  - Add "CSP Trusted Sites" section

#### reference/test-data-patterns.md
- **Current issue**: Factories and setup, but incomplete.
- **Fix**:
  - Add mermaid diagram: test data factory pattern (factory → creates hierarchy → returns for assertion)
  - "Test Data Factory": class structure, example
  - "Bulk Testing": create 200+ records efficiently
  - "System.runAs": create test user, assign PermissionSet, run as that user
  - "Assertions": assert all fields (not just one), assert error cases
  - Verify all syntax is valid Apex

### README.md (1 file)

#### README.md
- **Current issue**: Has role-based navigation (good), but no architecture diagram.
- **Fix**:
  - Add mermaid diagram at top: "Salesforce Development Ecosystem" showing how all components connect
  - Components in diagram: Triggers → Flows → LWC → Integration → Permissions → Testing
  - Each component links to relevant docs
  - "Getting Started" section: choose by role (Developer, Admin, Architect)
  - Remove any marketing language
  - Add "Facts About This Repo" section: what we cover, what we don't

---

## Session 2 Implementation Plan

**Time**: ~2-3 hours for all 14 files at methodical pace

1. **Fix core concepts in order** (1-2 files per 30 min, with mermaid diagrams):
   - FLOW_BEST_PRACTICES.md (30 min)
   - INTEGRATION.md (30 min)
   - SECURITY.md (30 min)
   - LWC_BEST_PRACTICES.md (20 min — mostly diagrams)
   - TESTING_STRATEGY.md (30 min)
   - PERMISSIONS.md (30 min)
   - AI_PITFALLS.md (30 min)
   - MULTITENANT.md (30 min)

2. **Fix reference guides** (20 min each):
   - sf-cli-cheatsheet.md (verify + organize)
   - soql-anti-patterns.md (diagrams + fixes)
   - batch-apex-patterns.md (diagrams + patterns)
   - lwc-security.md (expand + diagrams)
   - test-data-patterns.md (expand + diagrams)

3. **Update README.md** (20 min):
   - Add ecosystem diagram
   - Add "Facts About This Repo"
   - Cross-link to all 14 docs

4. **Commit & Push** (5 min)
   - `git add .`
   - `git commit -m "refactor: expand all docs to full Salesforce ecosystem, add mermaid diagrams, remove AI patterns"`
   - `git push origin main`

---

## Notes for Next Session

- **ORDER_OF_EXECUTION.md** is the gold standard for this project: comprehensive, multi-component perspective, mermaid diagrams, no AI filler
- **Mermaid diagrams** are critical: every file needs at least one (sequenceDiagram or flowchart)
- **Perspective shifts** ("From Apex:", "From LWC:", "From Flow:") help readers see the full picture
- **Guard against AI patterns**: if you see "It's important to note...", "This ensures...", "is responsible for..." — remove it
- **No em dashes** — use parentheses or split sentences
- **Facts only** — no "best practices" or "critical" without context

