# Vibe Coding in Salesforce: AI-Assisted Development with Guardrails

**TL;DR**: AI can speed up Salesforce development, but only when you give it tight context about platform constraints. This guide shows how to use AI effectively while respecting Salesforce's multi-tenant architecture, governor limits, and order of execution.

---

## What is Vibe Coding?

Vibe coding is iterative AI-assisted development. You describe what you want. AI generates a first pass. You refine. You repeat until it's right.

In Salesforce, it looks like this:

```
1. Prompt AI with minimal, scoped context
2. Get back boilerplate or scaffolding
3. Validate against Salesforce constraints
4. Refine and iterate
5. Commit
```

**Where vibe coding helps:**
- Writing trigger handler boilerplate
- Generating test class structure and bulk tests
- Creating selector queries (with your guidance)
- Building LWC component structure
- Debugging error messages (with your context)

**Where it fails hard:**
- Governor limits (AI can't count DML operations across nested calls)
- Hidden execution paths (flows, validation rules, platform events)
- Multi-object interactions (update Account → triggers Job → queues Batch)
- Org-specific logic (custom settings, permission sets, record types)
- Security (AI guesses at FLS checks instead of validating)

The goal: use AI to eliminate boilerplate, not to replace engineering judgment.

---

## Why Salesforce Execution Order Matters for AI

The biggest gap between AI-generated code and production-ready code is execution order.

**Here's the execution sequence:**
1. Trigger fires (before/after)
2. Validation rules run
3. Flows execute
4. Assignment rules, escalation rules, auto-response rules
5. DML completes
6. Processes (if enabled, legacy)

**Why AI misses this:**
Your prompt says "Update Account when Opportunity closes." AI writes the trigger code. But it doesn't know:
- A flow also updates Account.Amount__c
- A validation rule blocks if Amount__c is negative
- The trigger tries to set Amount__c = -100

Result: Production error. Flow ran first, validation sees negative amount, blocks the transaction.

**Real example:**
```
AI-generated trigger (bad):
trigger OpportunityTrigger on Opportunity {
  for (Opportunity opp : Trigger.new) {
    Account acc = [SELECT Id, Revenue__c FROM Account WHERE Id = :opp.AccountId];
    acc.Revenue__c += opp.Amount;
  }
  update accounts;  // Problem: Flow already updated this. Double-count.
}
```

**What the developer should have told the AI:**
"Update Account.Revenue__c. But Account has a flow that also updates Revenue__c. Account also has a validation rule checking Revenue__c > 0. Tell me the exact execution order and what could conflict."

---

## Safe Vibe Coding Patterns

These patterns keep AI-generated code from breaking in production.

### Pattern 1: Scoped Prompting

Give the AI one thing, not everything.

**Bad prompt:**
```
"I have a complex org with 50 objects. Build me a system to sync Accounts, Contacts, 
Opportunities, and Orders with our external CRM when anything changes."
```

The AI will generate a massive system that touches everything and misses half the edge cases.

**Good prompt:**
```
"I have an Account object with a custom field Job_Status__c.
Requirement: When Job_Status__c = 'Complete', set Account.Revenue__c = sum of related Opportunity amounts.
Constraints: No hardcoded IDs, use with sharing, WITH SECURITY_ENFORCED, 200-record bulk test.
Show me a trigger handler class (don't write the full trigger yet)."
```

**Why this works:**
- Single responsibility (one field, one action)
- Constraints are explicit (bulk test, sharing, etc.)
- AI can't get lost in complexity

### Pattern 2: Human Validation Layer

AI generates. You validate against Salesforce constraints before using it.

**Validation checklist:**
- [ ] SOQL queries: count them. Are any in loops? Do they have a WHERE clause?
- [ ] DML operations: Are they outside loops? Are they batched?
- [ ] FLS: Does the code use WITH SECURITY_ENFORCED or Security.stripInaccessible()?
- [ ] Sharing: Does the class use `with sharing`? Does it handle public access correctly?
- [ ] Hardcoded IDs: Search for `'0' or '001' or '501'`. Any found? Reject.
- [ ] Bulk test: Does the test class include a 200-record test? What about edge cases?
- [ ] Order of execution: Does this trigger interact with any flows, validation rules, or other triggers?

**Example:**
AI generates this selector:
```apex
public List<Account> getAccountsByIndustry(String industry) {
  return [SELECT Id, Name, Industry FROM Account WHERE Industry = :industry];
}
```

**Your validation:**
- No WITH SECURITY_ENFORCED. Add it.
- No FLS check. If this is exposed via LWC, add Security.stripInaccessible().
- Good: Parameterized query, outside loop.

### Pattern 3: Incremental Generation

Don't ask AI for the whole system. Build it piece by piece.

**Bad:**
```
"Write me a complete trigger handler system with handlers for Account, Contact, Opportunity, 
and Order with all business logic."
```

**Good:**
```
1. "Write me the trigger handler base class (handles recursion, bulk, security)"
2. "Write me one handler for Account (Revenue calculation only)"
3. "Write me the selector class for Account"
4. "Write me the test class for the Account handler (200-record bulk test)"
```

Each piece is small enough to validate. You catch issues early.

### Pattern 4: Context Memory Usage

Once you've debugged a pattern, reuse the fix.

**First time:** You fix a trigger recursion bug.
```apex
public class AccountTriggerHandler {
  private static Boolean isExecuting = false;
  
  public static void handle(List<Account> records) {
    if (isExecuting) return;
    isExecuting = true;
    // logic
  }
}
```

**Next time:** You tell the AI: "Use the recursion pattern from [PATTERNS.md](PATTERNS.md). Here's the base class structure. Build on it."

The AI now has the proven pattern. No re-debugging.

---

## Align AI Outputs with Salesforce Design Patterns

AI-generated code should fit into real Salesforce architecture, not fight it.

### The Trigger Handler Pattern

This is the foundation. AI should output handlers, not monolithic triggers.

**Bad trigger (what naive AI might write):**
```apex
trigger OpportunityTrigger on Opportunity (after update) {
  for (Opportunity opp : Trigger.new) {
    Opportunity oldOpp = Trigger.oldMap.get(opp.Id);
    if (opp.Amount != oldOpp.Amount) {
      Account acc = [SELECT Id, Revenue__c FROM Account WHERE Id = :opp.AccountId];
      acc.Revenue__c = acc.Revenue__c + (opp.Amount - oldOpp.Amount);
      update acc;
    }
  }
}
```

Problems:
- SOQL inside loop (101 query limit at 100 records)
- No DML bulking
- No recursion guard
- Hard to test
- No separation of concerns

**Corrected structure (what you should ask AI for):**
```apex
trigger OpportunityTrigger on Opportunity (after update) {
  OpportunityTriggerHandler.handle(Trigger.new, Trigger.oldMap);
}

public class OpportunityTriggerHandler {
  public static void handle(List<Opportunity> newOpps, Map<Id, Opportunity> oldMap) {
    // Collect Ids
    Set<Id> accountIds = new Set<Id>();
    for (Opportunity opp : newOpps) {
      Opportunity oldOpp = oldMap.get(opp.Id);
      if (opp.Amount != oldOpp.Amount) {
        accountIds.add(opp.AccountId);
      }
    }
    
    // Bulk query once
    Map<Id, Account> accountMap = new Map<Id, Account>(
      AccountSelector.getByIds(accountIds)
    );
    
    // Update in memory
    for (Opportunity opp : newOpps) {
      Opportunity oldOpp = oldMap.get(opp.Id);
      if (opp.Amount != oldOpp.Amount) {
        Account acc = accountMap.get(opp.AccountId);
        if (acc != null) {
          acc.Revenue__c += (opp.Amount - oldOpp.Amount);
        }
      }
    }
    
    // Bulk DML once
    update accountMap.values();
  }
}

public class AccountSelector {
  public static List<Account> getByIds(Set<Id> accountIds) {
    return [SELECT Id, Revenue__c FROM Account WHERE Id IN :accountIds WITH SECURITY_ENFORCED];
  }
}
```

**When prompting AI for this:**
```
"Here's a trigger handler base structure [paste PATTERNS.md section]. 
Build the OpportunityTriggerHandler following this pattern. 
The requirement is: update Account.Revenue__c when Opportunity.Amount changes.
Use AccountSelector for queries. Include recursion guard."
```

---

## Multi-Tenant and Security Constraints

Salesforce is multi-tenant. That means your code runs alongside other customers' code. AI needs to respect that.

### Governor Limits

AI often ignores these:
- 101 SOQL queries per transaction
- 50,000 DML records per transaction
- 6 MB heap size
- 10 seconds CPU time
- 120 seconds total time

**What goes wrong:**
```apex
// AI might write this:
public List<Account> updateAllAccounts() {
  List<Account> accounts = [SELECT Id, Name FROM Account];  // 1M rows
  for (Account acc : accounts) {
    acc.Name = 'Updated';  // 1M updates
  }
  update accounts;  // Hits DML limit
}
```

**How to guard against it:**
Tell the AI: "We have 500,000 Accounts. Write a batch class that processes 200 records at a time."

### Sharing and FLS

AI assumes all users see all data. They don't.

**Sharing rule example:**
Your org has two teams: East Coast and West Coast. Your sharing rules say: West Coast users only see West Coast Accounts.

**Bad AI code:**
```apex
public List<Account> getTopAccounts() {
  return [SELECT Id, Name, Revenue__c FROM Account ORDER BY Revenue__c DESC LIMIT 100];
}
// A West Coast user might see East Coast accounts if they shouldn't.
```

**Good code:**
```apex
public List<Account> getTopAccounts() {
  return [SELECT Id, Name, Revenue__c FROM Account WITH SECURITY_ENFORCED ORDER BY Revenue__c DESC LIMIT 100];
}
```

WITH SECURITY_ENFORCED respects sharing rules. It won't return rows the user shouldn't see.

**FLS (Field-Level Security) example:**
Your org has sensitive fields like Salary__c. You don't want junior developers to see them.

**Bad AI code:**
```apex
public Map<Id, Account> getAccountsByIndustry(String industry) {
  return new Map<Id, Account>(
    [SELECT Id, Name, Industry, Salary__c FROM Account WHERE Industry = :industry]
  );
}
// If Salary__c is hidden from the user, this silently strips it. The caller thinks they have it.
```

**Good code:**
```apex
public Map<Id, Account> getAccountsByIndustry(String industry) {
  List<Account> accounts = [SELECT Id, Name, Industry, Salary__c FROM Account 
    WHERE Industry = :industry WITH SECURITY_ENFORCED];
  return new Map<Id, Account>(Security.stripInaccessible(AccessType.READABLE, accounts).getRecords());
}
```

Security.stripInaccessible removes fields the user can't read. The caller knows what they got.

### Data Sensitivity

AI can't know what's sensitive.

**Bad prompt:**
```
"Give me all customer Account data for debugging."
```

**Good prompt:**
```
"Debug the query for Accounts in Industry = 'Finance'. 
Show me Id, Name, Industry only. No customer data or PII."
```

---

## Component Coverage: Where AI Helps and Where It Fails

### Apex

**AI excels:**
- Writing selector classes (bulked queries)
- Generating test classes with bulk tests
- Creating trigger handler structure
- Writing service layer boilerplate

**AI struggles:**
- Across-trigger interactions (doesn't know about other triggers on the object)
- Recursive flows (AI can't count nested updates)
- Complex SOQL with subqueries (often gets the syntax wrong)
- Security logic (assumes everything is readable)

**Best practice:** Use AI for scaffolding. You write the business logic.

### LWC (Lightning Web Component)

**AI excels:**
- HTML template structure
- Event handling boilerplate
- Component lifecycle hooks
- Basic CSS styling

**AI struggles:**
- Apex integration (generates calls that don't match your service layer)
- Error handling for callouts
- Performance optimization (doesn't know which queries are slow)
- Accessibility (forgets ARIA labels, focus management)

**Best practice:** AI generates the UI. You wire it to your Apex classes and validate the contract.

**Example:**
```
Bad AI result:
// LWC tries to call Apex method that doesn't exist
import { LightningElement, wire } from 'lwc';
import getAccounts from '@salesforce/apex/AccountService.getAccounts';

Good workflow:
1. You define the Apex service method signature
2. AI generates the LWC component that calls it
3. You verify the call matches the Apex signature
```

### Flow

**AI struggles hard here:**
- Doesn't know about other flows on the same object
- Can't see order of execution (flows run after triggers)
- Gets flow formula syntax wrong
- Creates complex flows that are hard to debug

**Best practice:** Use AI to generate flow steps if they're simple. But you write the business logic and verify no conflicts.

**Common mistake:**
```
User asks AI: "Build me a flow that updates Account when Opportunity changes."
AI builds a flow.
User deploys it.
Trigger also updates Account.
Production: double-counts, validation fails.
```

**Better approach:**
Use triggers for data operations. Use flows for notifications and approvals. Keep them separated.

### Aura (Legacy)

AI can generate Aura, but most projects are moving to LWC.

Use AI for LWC. For Aura, only if you must maintain existing components.

---

## How This Ties Back to This Repository

This repo exists to make vibe coding work better by giving AI better context.

### The Problem

Without context, every AI interaction starts from zero:

```
Session 1 (March): Debug SOQL limit → you explain "never query in loops"
Session 2 (June): New dev, same issue → you explain it again
Session 3 (September): Another developer, same mistake → explain again
```

### The Solution

Encode the lessons once. Reuse them forever.

**This repo does that through:**

1. **CLAUDE.md template** — Load this into your AI tool. It has your 20 hard rules. AI reads them every session.
2. **PATTERNS.md** — Copy verified patterns instead of starting from scratch. Recursion guard, bulk test structure, trigger handler base class.
3. **ARCHITECTURE.md** — When AI is confused about bulkification or security, link to the architecture guide. AI reads the reasoning.
4. **DEPLOYMENT.md** — Before deploying, check the 16-phase order. Catch conflicts early.
5. **reference/** folder — Quick lookup for governor limits, common errors, metadata structure.

### How to Use This Repo with AI

**Step 1: Load CLAUDE.md**
Copy [templates/CLAUDE.md](templates/CLAUDE.md) into your project. Tell your AI tool: "Load this file and follow it."

**Step 2: Reference Patterns**
When you prompt the AI, point it to the right pattern:
```
"Implement an Account trigger handler using the pattern in PATTERNS.md. 
The requirement is..."
```

**Step 3: Validate Against Architecture**
After AI generates code, validate it against [ARCHITECTURE.md](ARCHITECTURE.md):
- Bulkified? Check against rule #11.
- Uses with sharing? Check against rule #7.
- FLS enforced? Check against rule #12.

**Step 4: Use Reference as Guardrails**
When AI gets stuck:
- Query limit? Check [reference/governor-limits.md](reference/governor-limits.md)
- Deployment issue? Check [reference/deployment-order.md](reference/deployment-order.md)
- Error message? Check [reference/common-errors.md](reference/common-errors.md)

---

## The Practical Workflow: How to Vibe Code Safely

Here's the exact workflow that makes vibe coding work in Salesforce.

### Step 1: Identify the Change

Define the scope tightly.

```
Don't say: "Update my data model"
Do say: "Add a custom field Revenue__c to Account. 
         When an Opportunity is marked Won, 
         add its Amount to Account.Revenue__c"
```

### Step 2: Extract Minimal Context

Pull only what's relevant.

- Current Apex class you're modifying (or none if new)
- Related objects (Account, Opportunity in the example above)
- Existing selector classes or trigger handlers
- Any validation rules or flows that touch the same object
- Constraints from CLAUDE.md

```
Context to give AI:
- Opportunity has Amount, StageName, AccountId
- Account has Revenue__c (custom, number field)
- Account has a flow called "ValidateRevenue" that checks Revenue__c > 0
- Must use with sharing, WITH SECURITY_ENFORCED, bulkify for 200+ records
- Test must include 200-record bulk test
```

```
Context to NOT give AI:
- Your entire org structure
- Competitor information
- Customer data (PII, secrets, production data)
- Unrelated objects
```

### Step 3: Prompt the AI

Be specific about output.

```
"Write the OpportunityTriggerHandler.handle() method (not the trigger).
Use the base class from PATTERNS.md. 
The requirement is: when Opportunity.StageName = 'Closed Won', 
update Account.Revenue__c += Opportunity.Amount.

Constraints:
- Use with sharing
- Bulk SOQL and DML
- WITH SECURITY_ENFORCED
- No hardcoded IDs

Show me just the handler class. I'll write the selector separately."
```

### Step 4: Validate Against Salesforce Constraints

Before you use the code, run through the checklist:

- [ ] SOQL queries: Are they outside loops? Do they use WITH SECURITY_ENFORCED?
- [ ] DML: Are all updates done in bulk outside loops?
- [ ] Sharing: Does the class use `with sharing`?
- [ ] FLS: Are sensitive fields handled with Security.stripInaccessible()?
- [ ] Hardcoded IDs: Any found? Reject.
- [ ] Recursion: Is there a guard to prevent infinite loops?
- [ ] Bulk test: Does the test class have a 200-record test?
- [ ] Order of execution: Will this conflict with flows, validation rules, or other triggers?

### Step 5: Refine

Don't accept the first output. Iterate.

```
You: "Add recursion guard to prevent infinite trigger loops"
AI: [Adds guard]
You: "Good. Now write the bulk test class with 200 records"
AI: [Generates test]
You: [Run it locally] "Test passes. But add a case for negative Amount. How should that be handled?"
AI: [Refines test]
```

---

## Real Example: Debugging a Query Limit Bug

**Scenario:** Your Account trigger hits 101 queries in production.

### Without Context (Bad)

```
You: "My trigger is querying inside a loop. Fix it."
AI: "Use a map to batch your queries."
[AI rewrites]
You: Look at code. "Did you forget WITH SECURITY_ENFORCED?"
AI: "Oh yes, let me add that."
[AI adds it]
You: "And the test needs a 200-record bulk test."
AI: "Sure, I'll add that."
[AI adds it]

Result: 3 iterations. AI never would have thought of WITH SECURITY_ENFORCED on its own.
```

### With Context (Good)

```
You: Load CLAUDE.md into your AI tool. 
     "Fix the query limit bug in AccountTriggerHandler. 
      Rules to follow are in the loaded CLAUDE.md file."

AI reads CLAUDE.md, sees:
  - Rule #4: Never SOQL or DML inside loops
  - Rule #12: Always use WITH SECURITY_ENFORCED
  - Rule #9: 90%+ coverage plus 200-record bulk test

AI rewrites the handler, adds WITH SECURITY_ENFORCED, writes the bulk test.

You: Review code. "Perfect. One iteration."

Result: 1 iteration. AI had the rules upfront.
```

---

## Another Real Example: Flow Conflict

**Scenario:** You write a trigger to update Account.Status__c. Production breaks.

### Without Context (Bad)

```
You: "Write a trigger to set Account.Status__c = 'Active' when an Opportunity is Won."
AI: [Writes trigger]
You: Deploy to production.
Production: Conflict with existing flow. Two components try to update Status__c.
You: Rollback. Debug for an hour. Find the flow. Rewrite both.

Result: Unplanned rollback. Incident.
```

### With Context (Good)

```
You: "Write a trigger for this requirement. 
      But first, check if any flows, validation rules, or other triggers touch 
      Account.Status__c. List any potential conflicts."

AI checks your documentation (from ARCHITECTURE.md context you provided):
"Found: Flow 'ValidateAccountStatus' also updates Status__c. 
 Risk: Both trigger and flow run. Status__c updated twice. Could cause validation failure.
 Recommendation: Move this logic to the flow. Remove from trigger."

You: "Good catch. Let's put this in the flow instead."

Result: Conflict caught in code review, not production.
```

---

## When to Stop Using AI

Not every task benefits from AI.

**Use AI for:**
- Boilerplate (trigger handler structure, test scaffolding)
- Syntax (LWC event bindings, Flow formula syntax)
- First passes (generate, then validate)

**Don't use AI for:**
- Security decisions (FLS, sharing rules, org-wide defaults)
- Order of execution logic (AI can't see all interactions)
- Complex SOQL with subqueries (AI often gets the syntax wrong)
- Business logic that requires domain knowledge (your org's rules, not Salesforce's)

---

## Summary

Vibe coding in Salesforce works when you:

1. **Scope tightly.** One object, one change, minimal context.
2. **Load guardrails.** Use CLAUDE.md or .cursorrules so AI knows the rules.
3. **Reference patterns.** Point AI to proven code in PATTERNS.md.
4. **Validate always.** Run through the constraint checklist before using AI output.
5. **Iterate.** Accept that the first pass might need refinement.
6. **Know when to stop.** AI is good at boilerplate. You're good at architecture.

Use this repo as your context library. Load it into your AI tool. Reference it in your prompts. Let AI speed up the repetitive parts. Keep your judgment on the critical parts.

---

## Next Steps

1. Copy [templates/CLAUDE.md](templates/CLAUDE.md) into your Salesforce project.
2. Load it into your AI tool (Claude Code, Cursor, Copilot, etc.).
3. Read [ARCHITECTURE.md](ARCHITECTURE.md) to understand the 10 core rules.
4. Use [PATTERNS.md](PATTERNS.md) as scaffolding for your next component.
5. When you hit a bug, check [reference/common-errors.md](reference/common-errors.md).

---

**See also:**
- [ARCHITECTURE.md](ARCHITECTURE.md) — Core patterns and rules
- [PATTERNS.md](PATTERNS.md) — Reusable code examples
- [DEPLOYMENT.md](DEPLOYMENT.md) — Deployment order and CI/CD
- [LIMITATIONS_AND_STATE.md](LIMITATIONS_AND_STATE.md) — What's experimental and what gaps exist
