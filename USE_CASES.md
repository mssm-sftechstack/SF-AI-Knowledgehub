# Salesforce Use Cases: Memory-Driven Development

Three realistic scenarios showing how persistent context and memory systems reduce friction in Salesforce development.

---

## Use Case 1: Apex Debugging Memory

### The Problem

**Scenario**: Your trigger hits the 101 query limit in production. You spend 4 hours debugging:
- Trace log analysis to find the loop
- Refactor SOQL into a map-based pattern
- Rewrite dependent code to use the map
- Write a bulk test (200+ records) to catch it
- Deploy a hotfix

**The friction without memory:**
- Next project, 3 months later, a junior dev writes the same query-in-loop pattern
- You catch it in code review and explain it again
- 2-3 code review cycles to get it right
- No formal record of what you learned

**Current workflow without memory system:**
```
Project A (March)        Project B (June)         Project C (September)
Debugging hit limit  →   Same mistake again   →  Same mistake AGAIN
4 hours fix          →   2 code reviews        →  2 code reviews + hotfix
Learn lesson         →   Explain to junior dev →  Explain to new team member
No record kept       →   No record kept        →  Pattern never gets encoded
```

---

### How the System Helps

**Document what you learn**
After you fix a bug, save it to SESSION_STATE.md:
```markdown
## Debugging Session

Issue: Trigger hitting 101 query limit on bulk update
Root cause: SOQL in for-loop on trigger.new (200+ records)
Fix: Collect IDs first, query once, build map, iterate map
Time to fix: 4 hours
Impact: Production hotfix, broke SLA
Next project: Look for "for.*loop.*SOQL" pattern in PR review
```

**Load memory in the next project**
Next project, you load HANDOFF.md from the previous one. When you ask the AI to review trigger code, it checks against your documented failures first.

**AI enforces the pattern**
From Project 2 onward, when the AI sees SOQL it asks:
```
SOQL Checklist:
- Query inside a for loop? 
- Querying 200+ records?
- Could use a map instead?
```

It's not magic memory. It's structured patterns enforced by what you've learned.

---

### Example of Stored Memory

**In PROJECT_A/SESSION_STATE.md** (after hotfix deployed):
```markdown
# Session State — Project A

## Debugging Session: March 15, 2026

### Issue
Trigger hitting 101 query limit in production.
Batch update of 1,200 Opportunities → trigger executes 1,200 times
Each iteration queries Account (for auto-follow logic)
Result: 1,200 queries + rollback

### Root Cause
Handler code:
```apex
for (Opportunity opp : opportunities) {
  List<Account> accts = [SELECT Id FROM Account WHERE Id = :opp.AccountId];
  opp.Follow_Up_Date__c = accts[0].Last_Touch__c;
}
```

### Fix Applied
Refactor to bulkified pattern:
```apex
Set<Id> accountIds = new Set<Id>();
for (Opportunity opp : opportunities) {
  accountIds.add(opp.AccountId);
}
Map<Id, Account> accountMap = new Map<Id, Account>(
  [SELECT Id, Last_Touch__c FROM Account WHERE Id IN :accountIds
   WITH SECURITY_ENFORCED]
);
for (Opportunity opp : opportunities) {
  if (accountMap.containsKey(opp.AccountId)) {
    opp.Follow_Up_Date__c = accountMap.get(opp.AccountId).Last_Touch__c;
  }
}
```

### Impact
- Reduced queries from 1,200 to 2
- Deploy time: 4 hours (hotfix)
- Detection: Should have been caught in PR review with a 200-record bulk test

### Transferable Learning
Lesson: Always use a bulk test (200+ records) for triggers.
Pattern: Collect IDs → Query once → Map → Iterate.
Guardrail to enforce: "No SOQL inside a for loop" (already in CLAUDE.md)

### For Next Project
- When AI generates trigger handler code, ask: "Does this pass a 200-record bulk test?"
- If reviewer sees SOQL inside a loop, reject immediately.
```

**In HANDOFF.md** (passed to next session/developer):
```markdown
## Recent Debugging Lessons

### Query Limit Incident (March 15)
A trigger queried inside a for-loop. Caught in production, cost 4 hours to fix.
Next time: Bulk test before code review. 200-record test would have caught it.

### How to spot it:
- Grep for: `for.*in.*\[SELECT`
- Review checklist: Does this pass a 200-record bulk test?
```

---

### Example of Improved Future Output

**Project B (June) — Same Task, Different Outcome**

Developer writes a similar trigger. **With memory, the AI catches it upfront:**

```
Developer: "Here's my Account trigger that updates Related_Opportunity_Count.
Can you review it?"

AI (with PROJECT_A memory loaded):
❌ HALT. This query pattern failed in Project A.

Current code:
for (Account acc : accounts) {
  List<Opportunity> opps = [SELECT Id FROM Opportunity 
                             WHERE AccountId = :acc.Id];
  acc.Related_Opportunity_Count__c = opps.size();
}

Issue: On a 200+ record bulk operation, this hits query limits.

Fixed version (pattern from Project A):
Set<Id> accountIds = new Set<Id>();
for (Account acc : accounts) {
  accountIds.add(acc.Id);
}
Map<Id, List<Opportunity>> oppsByAccount = new Map<Id, List<Opportunity>>();
for (Opportunity opp : [SELECT Id, AccountId FROM Opportunity 
                        WHERE AccountId IN :accountIds
                        WITH SECURITY_ENFORCED]) {
  if (!oppsByAccount.containsKey(opp.AccountId)) {
    oppsByAccount.put(opp.AccountId, new List<Opportunity>());
  }
  oppsByAccount.get(opp.AccountId).add(opp);
}
for (Account acc : accounts) {
  acc.Related_Opportunity_Count__c = 
    oppsByAccount.containsKey(acc.Id) ? oppsByAccount.get(acc.Id).size() : 0;
}

Test: This code passes a 200-record bulk test. ✅
```

**Result**: Pattern caught before code review. No iteration cycle. Same fix, 10 minutes vs 2 hours.

---

## Use Case 2: SOQL Query Optimization Tracking

### The Problem

Your LWC is slow. You spend 2 hours optimizing:
- Start with SELECT * (50+ fields, slow)
- Narrow to just the fields you need
- Add indexed filters
- Use aggregate instead of loops
- Test at scale

Then nothing gets written down. Three months later, someone else writes the exact same slow query. You help them optimize. Same lessons. Same time spent.

Six months later, it happens again. Different team, different feature, same slow pattern.

---

### How the System Helps

**Document what you learn**
When you optimize a query, save it to SOQL_PATTERNS.md:
- What was slow (500ms)
- Why (unindexed filter)
- What fixed it (added indexed filter)
- The pattern you can reuse anywhere

**Build a checklist**
Next project, the AI has your checklist:
```
SOQL Performance (from your past projects):
- Select only fields you use (not *)
- Filter on indexed fields (Date, Status, lookup)
- Use aggregate for COUNT/SUM, not loops
- Test with 100k+ records, not 50
```

**Prevent the same mistakes**
- AI won't generate SELECT * without a reason
- AI suggests indexed filters by default
- AI warns when loading 10k+ records

---

### Example of Stored Memory

**In a shared SOQL_PATTERNS.md** (team-level memory):

```markdown
# SOQL Performance Lessons from Real Projects

## Lesson 1: Unnecessary Field Bloat

### Original Query (Project A, April)
```apex
List<Opportunity> opps = [SELECT * FROM Opportunity WHERE AccountId = :accId];
```
**Symptom**: LWC loading slowly (500ms)
**Root cause**: Fetching 60+ fields, only needed 3 (Name, Amount, StageName)
**Fix**: 
```apex
List<Opportunity> opps = [SELECT Id, Name, Amount, StageName 
                           FROM Opportunity 
                           WHERE AccountId = :accId
                           WITH SECURITY_ENFORCED];
```
**Performance**: 500ms → 80ms (6x faster)
**Rule**: SELECT only the fields you use. Even SELECT Id, Name, Amount matters at scale.

## Lesson 2: Unindexed Filters

### Slow Query (Project B, August)
```apex
List<Case> cases = [SELECT Id, Subject FROM Case 
                    WHERE Description LIKE :searchTerm];
```
**Symptom**: 2,000+ record query taking 1,200ms
**Root cause**: Description field is not indexed. LIKE scan is O(n).
**Why it happened**: Added a search feature without checking field indexes.
**Fix**: Query indexed fields instead:
```apex
List<Case> cases = [SELECT Id, Subject FROM Case 
                    WHERE Status = 'Open'
                    AND CreatedDate = LAST_N_DAYS:30
                    WITH SECURITY_ENFORCED
                    LIMIT 1000];
```
**Performance**: 1,200ms → 50ms (24x faster)
**Rule**: Filter on indexed fields (Status, CreatedDate, Lookup, Master-Detail). Check org setup before writing queries.

## Lesson 3: N+1 in a Loop

### Original Pattern (Project A, April)
```apex
for (Account acc : accounts) {
  List<Opportunity> opps = [SELECT COUNT() FROM Opportunity WHERE AccountId = :acc.Id];
  acc.Opp_Count__c = opps.size();  // Off by 1 error here
}
```
**Symptom**: 500 accounts = 501 queries. Hits limit at 400 accounts.
**Root cause**: Counting in a loop instead of aggregating.
**Fix**: Use aggregate:
```apex
Map<Id, AggregateResult> results = new Map<Id, AggregateResult>();
for (AggregateResult ar : [SELECT AccountId, COUNT(Id) cnt 
                           FROM Opportunity 
                           WHERE AccountId IN :accountIds
                           GROUP BY AccountId
                           WITH SECURITY_ENFORCED]) {
  results.put((Id)ar.get('AccountId'), ar);
}
for (Account acc : accounts) {
  AggregateResult ar = results.get(acc.Id);
  acc.Opp_Count__c = ar != null ? (Integer)ar.get('cnt') : 0;
}
```
**Performance**: 501 queries → 2 queries
**Rule**: Never COUNT in a loop. Use GROUP BY + aggregate.

## Transferable Pattern: The SOQL Optimization Checklist

Before writing any production SOQL:
1. ✅ SELECT only the fields you need (not *)
2. ✅ Filter on indexed fields (Status, CreatedDate, lookup, master-detail)
3. ✅ Use aggregate() for COUNT, SUM, MAX, MIN (not manual loops)
4. ✅ Use LIMIT + pagination for large datasets (>10k records)
5. ✅ WITH SECURITY_ENFORCED on every query
6. ✅ Test with real data volumes (100k+ if prod has it), not 50-record dev samples

## When NOT to Optimize (Over-Optimization Anti-Patterns)

- **Don't SELECT only 1 field when you need 5**: Pick what's actually needed, stop at "good enough"
- **Don't create manual indexes for fringe cases**: Talk to architect first
- **Don't build pagination for 300-record result sets**: Page when >10k results
```

**In PROJECT_B/SESSION_STATE.md** (after optimization):
```markdown
## Query Optimization Session (August 12)

### Query Performance Issue
LWC showing case list, loading slowly (800ms).

### Investigation
- Initial query: SELECT * (60 fields)
- Applied rule from SOQL_PATTERNS.md
- Narrowed to: SELECT Id, Subject, Status, CreatedDate (4 fields)
- Added indexed filter: Status = 'Open' AND CreatedDate = LAST_N_DAYS:30

### Result
- 800ms → 120ms (6x faster)
- Applied pattern from Project A (April) without re-optimizing from scratch

### Lessons Reinforced
- Field count matters (tested: * vs 60 fields vs 4 fields)
- Indexed filter selection is critical
```

---

### Example of Improved Future Output

**Project C (December) — New Query, Pattern-Aware**

Developer asks: "Write a query to find all Contacts modified in the last 90 days."

**Without memory:**
```apex
List<Contact> contacts = [SELECT * FROM Contact 
                          WHERE LastModifiedDate = LAST_N_DAYS:90];
```

**With memory (SOQL_PATTERNS.md loaded):**
```apex
// AI applies checklist from past projects:

// ✅ Rule 1: SELECT specific fields, not *
// ✅ Rule 2: Filter on indexed field (LastModifiedDate is indexed)
// ✅ Rule 3: Add LIMIT for safety
// ✅ Rule 4: Add WITH SECURITY_ENFORCED

List<Contact> contacts = [
  SELECT Id, FirstName, LastName, Email, Phone, AccountId
  FROM Contact 
  WHERE LastModifiedDate = LAST_N_DAYS:90
  WITH SECURITY_ENFORCED
  LIMIT 50000
];

// If you need more than 50k records, use pagination:
// Checkpoint: This query pattern performs at scale.
// If results exceed LIMIT, use queryMore() or add CreatedDate filter.
```

**Result**: Optimized query written on first pass. No slow code reaches review. Query is FLS-safe and index-aware by default.

---

## Use Case 3: Trigger Pattern Reuse

### The Problem

You build account triggers in three different projects:
- Project A: Auto-follow logic (before and after update)
- Project B: Sync to external system (async, after update)
- Project C: Duplicate validation (before insert)

Each time, you structure it differently. Handler classes have different names. You debate Queueable vs Platform Events again. You test differently each time. You hit the same order-of-execution gotchas and debug them again.

By Project C, you realize you should have built one template in Project A and reused it. But now the pattern is split across three projects with three different approaches.

---

### How the System Helps

**Create a template**
After Project A, save TRIGGER_PATTERNS.md with:
- Handler structure (how to organize before-insert vs after-update)
- When to use Queueable vs Platform Events
- Testing patterns (bulk tests, mock callouts)
- Order-of-execution gotchas

**Use it in the next project**
Project B: "Create an account trigger for syncing to external system on update"

AI applies TRIGGER_PATTERNS.md:
- Uses your handler structure
- Recommends Queueable (since it's a callout)
- Bulkifies automatically (Map, not loops)
- Generates a 200-record bulk test
- Documents assumptions (API version, endpoints, auth)

**Naming is consistent**
No more mixing AccountHandler, AccountTriggerHelper, Account_Trigger_Handler. One naming convention across all projects.

---

### Example of Stored Memory

**In TRIGGER_PATTERNS.md** (team-level, reusable):

```markdown
# Trigger Design Patterns from Production Orgs

## Pattern 1: Before-Insert Validation (Project A, January)

### Use Case
Validate Account before insert. Example: prevent duplicate accounts by unique name.

### Handler Structure
```apex
public with sharing class Account_Before_Insert_Handler {
  
  public static void handle(List<Account> records) {
    validateNoDuplicates(records);
    enrichDefaults(records);
  }
  
  private static void validateNoDuplicates(List<Account> records) {
    Set<String> names = new Set<String>();
    for (Account acc : records) {
      names.add(acc.Name);
    }
    Map<String, Account> existing = new Map<String, Account>(
      [SELECT Name FROM Account WHERE Name IN :names]
    );
    for (Account acc : records) {
      if (existing.containsKey(acc.Name)) {
        acc.addError('Account with this name already exists');
      }
    }
  }
}
```

### Key Points
- `with sharing` enforces FLS
- Query is bulkified (Map pattern, not loop)
- addError() blocks insert at trigger level (no need for separate validation)
- Test: 200-record insert with 50 duplicates

### When to Use
Before-insert for: validation, defaults, cross-field dependencies.
NOT for: callouts (use Queueable after-insert instead)

---

## Pattern 2: After-Update Async (Project B, May)

### Use Case
Sync Account updates to external system (callout) without blocking the trigger.

### Trigger Structure
```apex
trigger Account_Trigger on Account (after update) {
  Account_After_Update_Handler.handle(Trigger.newMap, Trigger.oldMap);
}
```

### Handler Structure
```apex
public with sharing class Account_After_Update_Handler {
  
  public static void handle(Map<Id, Account> newMap, Map<Id, Account> oldMap) {
    List<Account> changedAccounts = new List<Account>();
    for (Account newAcc : newMap.values()) {
      Account oldAcc = oldMap.get(newAcc.Id);
      if (newAcc.Sync_Field__c != oldAcc.Sync_Field__c) {
        changedAccounts.add(newAcc);
      }
    }
    if (!changedAccounts.isEmpty()) {
      System.enqueueJob(new Account_Sync_Queueable(changedAccounts));
    }
  }
}
```

### Queueable Class
```apex
public with sharing class Account_Sync_Queueable 
  implements Queueable, Database.AllowsCallouts {
  
  private List<Account> accounts;
  
  public Account_Sync_Queueable(List<Account> accounts) {
    this.accounts = accounts;
  }
  
  public void execute(QueueableContext ctx) {
    for (Account acc : accounts) {
      externalSystemClient.syncAccount(acc);
    }
  }
}
```

### Key Points
- Trigger NEVER makes callout (use Queueable)
- Database.AllowsCallouts on Queueable (allows callout)
- Handler is bulkified (collects changed records, queues once)
- Test: Mock the external system client, assert Queueable is enqueued

### When to Use
After-update + Queueable for: external API calls, heavy processing
NOT for: simple field updates (use before-insert or flows instead)

---

## Pattern 3: Conflicting Async Patterns (Project C, October)

### The Problem
Two features both need to react to Account updates:
- Feature A: Sync to external system (after-update, Queueable with callout)
- Feature B: Update rollup count (after-update, synchronous calculation)

### The Anti-Pattern (Do NOT do this)
```apex
// ❌ WRONG: Two triggers on same object
trigger Account_Sync_Trigger on Account (after update) { ... }
trigger Account_Rollup_Trigger on Account (after update) { ... }
// Problem: Execution order is undefined. One may fail before other runs.
// Hard rule: ONE trigger per object.
```

### The Correct Pattern
```apex
trigger Account_Trigger on Account (after update) {
  Account_After_Update_Handler.handle(Trigger.newMap, Trigger.oldMap);
}

public with sharing class Account_After_Update_Handler {
  
  public static void handle(Map<Id, Account> newMap, Map<Id, Account> oldMap) {
    List<Account> changedAccounts = new List<Account>();
    List<Account> rollupAccounts = new List<Account>();
    
    for (Account newAcc : newMap.values()) {
      Account oldAcc = oldMap.get(newAcc.Id);
      
      if (newAcc.External_Sync_Field__c != oldAcc.External_Sync_Field__c) {
        changedAccounts.add(newAcc);
      }
      if (newAcc.Opportunity_Amount__c != oldAcc.Opportunity_Amount__c) {
        rollupAccounts.add(newAcc);
      }
    }
    
    if (!changedAccounts.isEmpty()) {
      System.enqueueJob(new Account_Sync_Queueable(changedAccounts));
    }
    
    if (!rollupAccounts.isEmpty()) {
      Account_Rollup_Service.calculateRollups(rollupAccounts);
    }
  }
}
```

### Key Points
- ONE trigger, ONE handler
- Handler routes to different services (Sync, Rollup)
- Execution order is guaranteed: code runs sequentially
- Both features are bulkified and testable independently

### When to Use This Pattern
Multiple features on same object → extend the handler, never create a second trigger

---

## Order-of-Execution Gotchas

### Gotcha 1: Subflow Triggers Handler AGAIN
If your after-insert handler calls a flow, and the flow updates the same record, the trigger fires again (if it's update-triggered).

**Solution**: Use static flag to prevent recursion
```apex
public with sharing class Account_After_Insert_Handler {
  private static Boolean hasRun = false;
  
  public static void handle(List<Account> records) {
    if (hasRun) return;
    hasRun = true;
    
    // Safe to call flow here
    Map<String, Object> flowInputs = new Map<String, Object>();
    flowInputs.put('accountIds', new List<Id>(/* ... */));
    Flow.Interview.My_Flow flow = new Flow.Interview.My_Flow(flowInputs);
    flow.start();
  }
}
```

### Gotcha 2: Rollup Triggers Another Trigger
If you use a rollup summary field (master-detail), updating the detail record triggers the master's trigger.

**Solution**: Document this dependency. Test with related records.

### Gotcha 3: Platform Events Fire After Trigger
If your trigger publishes a Platform Event, subscribers fire *after* the trigger completes.

**Solution**: Use Platform Events for async work that happens *after* the transaction (audit, external notifications). Use Queueable for work that must happen *within* the same transaction.

---

## Trigger Testing Pattern

All triggers use this test structure:

```apex
@IsTest
private class Account_Trigger_Test {
  
  @IsTest
  static void testBulkInsert200Records() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
      accounts.add(new Account(Name = 'Test ' + i));
    }
    
    Test.startTest();
    insert accounts;
    Test.stopTest();
    
    List<Account> result = [SELECT Id FROM Account WHERE Name LIKE 'Test %'];
    Assert.areEqual(200, result.size());
  }
  
  @IsTest
  static void testCalloutWithMock() {
    Test.setMock(HttpCalloutMock.class, new ExternalSystemMock());
    
    Account acc = new Account(Name = 'Test', Sync_Field__c = 'value');
    
    Test.startTest();
    insert acc;
    Test.stopTest();
    
    // Assert Queueable was called (check logs or mock invocation count)
  }
}
```

### Key Points
- 200-record bulk test (catches governor limit issues)
- Callout test with mock (no real HTTP calls)
- Async test with Test.startTest/stopTest (ensures Queueable runs)
- 90%+ coverage minimum

---

## Trigger Checklist

Before deploying any trigger:
- [ ] ONE trigger per object
- [ ] Handler class (not inline logic in trigger)
- [ ] Bulkified (Map pattern, not loops)
- [ ] No callouts from trigger (use Queueable)
- [ ] No hardcoded IDs or org-specific values
- [ ] WITH SECURITY_ENFORCED in SOQL
- [ ] `with sharing` on all classes
- [ ] 200-record bulk test passes
- [ ] 90%+ coverage minimum
- [ ] Order-of-execution documented (if calling flows, publishing events, etc.)
```

**In PROJECT_C/SESSION_STATE.md** (after deploying trigger):
```markdown
## Trigger Implementation (October 8)

### Requirement
Validate Account before insert: prevent duplicates by name.

### Pattern Used
TRIGGER_PATTERNS.md → Pattern 1: Before-Insert Validation
- Handler class: Account_Trigger_Handler (reused from Projects A & B)
- Structure: Trigger → Handler → Service
- Test: 200-record bulk insert with 50 duplicates

### Customizations Made
- Added check for external system IDs (org-specific)
- Enriched defaults (Billing Address from Shipping, if null)

### Result
- Deployment time: 30 minutes (most time spent in testing, not building structure)
- Code review: 1 iteration (already familiar with pattern)
- Test coverage: 94%

### Lessons Reinforced
- Template approach saves dev time
- Pattern consistency helps code review
```

---

### Example of Improved Future Output

**Project D (Next Year) — Pattern Library Mature**

Developer: "I need a trigger that updates the annual contract value when an Opportunity closes."

**AI (with TRIGGER_PATTERNS.md):**

```
✅ Pattern Match: After-Update Synchronous Calculation
(Similar to Pattern 2, but synchronous not async)

Generated trigger:
```apex
trigger Opportunity_Trigger on Opportunity (after update) {
  Opportunity_After_Update_Handler.handle(Trigger.newMap, Trigger.oldMap);
}

public with sharing class Opportunity_After_Update_Handler {
  public static void handle(Map<Id, Opportunity> newMap, Map<Id, Opportunity> oldMap) {
    List<Opportunity> closedOpps = new List<Opportunity>();
    
    for (Opportunity newOpp : newMap.values()) {
      Opportunity oldOpp = oldMap.get(newOpp.Id);
      if (newOpp.StageName != oldOpp.StageName && newOpp.StageName == 'Closed Won') {
        closedOpps.add(newOpp);
      }
    }
    
    if (!closedOpps.isEmpty()) {
      Opportunity_ACV_Service.updateACV(closedOpps);
    }
  }
}
```

Generated service class:
```apex
public with sharing class Opportunity_ACV_Service {
  public static void updateACV(List<Opportunity> opportunities) {
    Set<Id> accountIds = new Set<Id>();
    for (Opportunity opp : opportunities) {
      accountIds.add(opp.AccountId);
    }
    
    // Bulkified query
    Map<Id, Account> accounts = new Map<Id, Account>(
      [SELECT Id, Annual_Contract_Value__c FROM Account 
       WHERE Id IN :accountIds
       WITH SECURITY_ENFORCED
       FOR UPDATE]
    );
    
    for (Opportunity opp : opportunities) {
      Account acc = accounts.get(opp.AccountId);
      if (acc != null) {
        acc.Annual_Contract_Value__c = 
          (acc.Annual_Contract_Value__c ?? 0) + opp.Amount;
      }
    }
    
    update accounts;
  }
}
```

✅ Structure follows established pattern from Projects A-C
✅ Bulkified (no loops, Map usage, bulk update)
✅ Test template included (200-record bulk test)
✅ Naming convention consistent (Opportunity_After_Update_Handler)

Ready to code review.
```

**Result**: Trigger template generated correctly on first pass. Code review familiar with pattern. Deploy in 20 minutes. No order-of-execution surprises because pattern is proven.

---

## The Meta Pattern: How Memory Accelerates Development

All three use cases follow the same flow:

| Phase | Without Memory | With Memory |
|---|---|---|
| **Project A** | Learn hard way (4 hours) | Document it |
| **Project B** | Re-learn (2 hours) | Apply pattern (15 min) |
| **Project C** | Teach someone (1 hour) | Follow checklist (10 min) |
| **Project D+** | Pattern stays tribal | Pattern is formalized |

Time saved per project: 2-3 hours (mid-size features). Across 10 projects, that's 20-30 hours plus better code.

**When this works:**
- You document lessons right after debugging
- Patterns live in git (PATTERNS.md, SESSION_STATE.md, HANDOFF.md)
- New projects load old lessons at startup
- AI templates enforce patterns in every prompt

**When this breaks:**
- Lessons stay in Slack (lost in 3 months)
- Patterns are verbal (told to new devs, not written)
- Teams don't share memory (each project starts fresh)
- Templates get stale (Salesforce changes, pattern becomes obsolete)

---

## How to Start

1. Pick one real issue you debugged recently (governor limit, slow query, trigger bug).
2. Write it down in LESSON.md (problem, fix, pattern you can reuse).
3. Add it to CLAUDE.md or .cursorrules for your next project.
4. Measure: Did code review catch fewer issues? Did dev time drop?
5. Share what works in CONTRIBUTING.md.

Start with one lesson. Over time, this becomes your org's playbook.
