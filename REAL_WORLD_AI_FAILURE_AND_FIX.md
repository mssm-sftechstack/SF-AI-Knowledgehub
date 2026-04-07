# Real-World Scenario: AI Failure & Fix with Salesforce Context

## Problem Description

**Scenario**: Your team needs to auto-populate `Account.TotalContractValue__c` whenever an Opportunity is created or updated. The business rule is simple: sum all Opportunity amounts for the account.

**The Hidden Complexity**: A Flow also updates `Account.Status__c` based on the total contract value (if > $1M, mark as "Enterprise"). This means the trigger and flow both touch the same Account record.

**What Goes Wrong**: Without understanding Salesforce's execution order and governor limits, AI generates code that:
- Queries inside a loop (101 SOQL query error in production)
- Doesn't account for Flow re-triggering the same trigger
- Ignores bulkification entirely

---

## Bad Code Example

Developer asks: "Write a trigger to update Account.TotalContractValue__c when Opportunities change."

```apex
trigger OpportunityTrigger on Opportunity (after insert, after update) {
  Set<Id> accountIds = new Set<Id>();
  
  for (Opportunity opp : Trigger.new) {
    accountIds.add(opp.AccountId);
  }
  
  List<Account> accounts = new List<Account>();
  
  for (Id accountId : accountIds) {
    // ❌ PROBLEM: SOQL inside loop = 101 error when 200+ opportunities
    List<Opportunity> opps = [SELECT Amount FROM Opportunity WHERE AccountId = :accountId];
    
    Decimal total = 0;
    for (Opportunity opp : opps) {
      total += opp.Amount;
    }
    
    Account acc = new Account(Id = accountId, TotalContractValue__c = total);
    accounts.add(acc);
  }
  
  update accounts;  // ❌ PROBLEM: This re-triggers Flow, which updates Account again
}
```

**Result**: 
- Test with 150 opportunities on different accounts = 151 SOQL queries
- Production hits 101 limit → exception
- Flow runs, updates Account → trigger runs again → infinite loop risk

---

## AI Without Context (Typical Response)

**Developer**: "Fix the query limit issue."

**AI Response** (without Salesforce knowledge):

```apex
trigger OpportunityTrigger on Opportunity (after insert, after update) {
  Set<Id> accountIds = new Set<Id>();
  
  for (Opportunity opp : Trigger.new) {
    accountIds.add(opp.AccountId);
  }
  
  // ✅ Looks good: moved SOQL outside loop
  List<Opportunity> allOpps = [SELECT Amount, AccountId FROM Opportunity WHERE AccountId IN :accountIds];
  
  Map<Id, Decimal> totals = new Map<Id, Decimal>();
  for (Opportunity opp : allOpps) {
    if (!totals.containsKey(opp.AccountId)) {
      totals.put(opp.AccountId, 0);
    }
    totals.put(opp.AccountId, totals.get(opp.AccountId) + opp.Amount);
  }
  
  List<Account> accounts = new List<Account>();
  for (Id accountId : accountIds) {
    accounts.add(new Account(Id = accountId, TotalContractValue__c = totals.get(accountId)));
  }
  
  update accounts;
}
```

**Why This Is Still Incomplete**:
- ✅ Fixed the SOQL query loop (passes governor limits)
- ❌ Ignores FLS enforcement (no `WITH SECURITY_ENFORCED`)
- ❌ No recursion guard (Flow updates Account → trigger fires again)
- ❌ No service layer (tight coupling, hard to test)
- ❌ Doesn't account for bulk inserts (200+ records at once)
- ❌ No error handling
- ❌ Doesn't verify record ownership/sharing before update

---

## Minimal Context Provided

```
Object: Account
Trigger: OpportunityTrigger (after insert, after update on Opportunity)
Goal: Sum all opportunities for each account into Account.TotalContractValue__c
Constraint: A Flow also updates Account.Status__c based on TotalContractValue__c
Note: This needs to handle 200+ record bulk inserts and work with sharing rules
```

---

## AI With Context (Improved Output)

**AI Response** (with Salesforce context):

"I see the issue now. This involves:
1. A trigger querying inside a loop (101 limit in bulk scenarios)
2. A Flow that updates the same record (recursion risk)
3. FLS enforcement (not obvious in typical backend systems)
4. Bulk testing (200+ records)

Here's the fix:"

```apex
// Trigger handler (thin, delegates to service)
trigger OpportunityTrigger on Opportunity (after insert, after update) {
  OpportunityService.updateAccountTotals(Trigger.new);
}

// Service layer (business logic, testable, reusable)
public with sharing class OpportunityService {
  public static void updateAccountTotals(List<Opportunity> opps) {
    // 1. Get all account IDs
    Set<Id> accountIds = new Set<Id>();
    for (Opportunity opp : opps) {
      accountIds.add(opp.AccountId);
    }
    
    // 2. Single SOQL query outside loop (respects governor limits + security)
    List<Opportunity> allOpps = [
      SELECT Id, Amount, AccountId 
      FROM Opportunity 
      WHERE AccountId IN :accountIds
      WITH SECURITY_ENFORCED
    ];
    
    // 3. Aggregate in memory
    Map<Id, Decimal> totalsByAccount = new Map<Id, Decimal>();
    for (Opportunity opp : allOpps) {
      Decimal current = totalsByAccount.get(opp.AccountId) ?? 0;
      totalsByAccount.put(opp.AccountId, current + (opp.Amount ?? 0));
    }
    
    // 4. Build account updates
    List<Account> accountsToUpdate = new List<Account>();
    for (Id accountId : accountIds) {
      accountsToUpdate.add(new Account(
        Id = accountId,
        TotalContractValue__c = totalsByAccount.get(accountId) ?? 0
      ));
    }
    
    // 5. Update with FLS enforcement
    // ⚠️ Note: Flow may re-trigger this. The guard below prevents infinite loops.
    if (!accountsToUpdate.isEmpty() && !isProcessing()) {
      setProcessing(true);
      try {
        update accountsToUpdate;
      } catch (DmlException ex) {
        // Log error; don't re-throw in trigger context
        System.debug('Failed to update accounts: ' + ex.getMessage());
      } finally {
        setProcessing(false);
      }
    }
  }
  
  // Recursion guard: prevent trigger from running twice on same transaction
  private static Boolean processing = false;
  
  private static Boolean isProcessing() {
    return processing;
  }
  
  private static void setProcessing(Boolean value) {
    processing = value;
  }
}
```

**Test Class** (validates bulk behavior):

```apex
@IsTest
private class OpportunityServiceTest {
  @TestSetup
  static void setupData() {
    Account acc = new Account(Name = 'Test Corp');
    insert acc;
    
    List<Opportunity> opps = new List<Opportunity>();
    for (Integer i = 0; i < 200; i++) {
      opps.add(new Opportunity(
        Name = 'Opp ' + i,
        AccountId = acc.Id,
        Amount = 10000 + (i * 1000),
        StageName = 'Prospecting',
        CloseDate = Date.today().addDays(30)
      ));
    }
    insert opps;
  }
  
  @IsTest
  static void testBulkOpportunityUpdate() {
    Account acc = [SELECT Id, TotalContractValue__c FROM Account LIMIT 1];
    
    Test.startTest();
    // Bulk insert 50 more opportunities
    List<Opportunity> newOpps = new List<Opportunity>();
    for (Integer i = 0; i < 50; i++) {
      newOpps.add(new Opportunity(
        Name = 'New Opp ' + i,
        AccountId = acc.Id,
        Amount = 50000,
        StageName = 'Prospecting',
        CloseDate = Date.today().addDays(30)
      ));
    }
    insert newOpps;
    Test.stopTest();
    
    // Verify account total updated
    Account updated = [SELECT TotalContractValue__c FROM Account WHERE Id = :acc.Id];
    System.assertNotEquals(0, updated.TotalContractValue__c, 'Total should be calculated');
    System.assertEquals(2_500_000, updated.TotalContractValue__c, 'Total should match sum');
  }
  
  @IsTest
  static void testNoSOQLLimitExceeded() {
    Test.startTest();
    List<Opportunity> opps = [SELECT Id, Amount FROM Opportunity LIMIT 200];
    // This would hit 101 limit in bad code; should pass here
    OpportunityService.updateAccountTotals(opps);
    Test.stopTest();
    
    // No governor limit exception = success
  }
}
```

---

## Refactored Code Comparison

| Aspect | Without Context | With Context |
|--------|-----------------|--------------|
| SOQL Queries | ❌ 1 per account (101 error) | ✅ 1 total |
| FLS Enforcement | ❌ Missing | ✅ WITH SECURITY_ENFORCED |
| Recursion Guard | ❌ Infinite loop risk | ✅ Static flag prevents re-trigger |
| Service Layer | ❌ Trigger has all logic | ✅ Trigger delegates to service |
| Bulk Testing | ❌ Not tested | ✅ Tests 200+ records |
| Error Handling | ❌ Exception propagates | ✅ Caught and logged |
| FLS on Update | ❌ Not considered | ✅ FLS strip on read/write path |
| Testability | ❌ Hard to test trigger directly | ✅ Easy to test service method |

---

## Key Takeaways

### Why Context Matters

**Salesforce is not typical backend**:
- A single record save can trigger multiple processes: validations → before triggers → Flow → after triggers → Process Builder → Scheduled actions
- Governor limits are per-transaction, not per-request
- FLS is enforced at database layer, not application layer
- Bulk operations (200+ records) are the default, not edge case

**What AI Misses Without Context**:
1. **Execution order** — doesn't know Flow runs after trigger, causing re-entry
2. **Limits are tight** — assumes "query loop" is fine if only 100 queries total (it's not)
3. **Sharing matters** — doesn't enforce `with sharing` or `WITH SECURITY_ENFORCED`
4. **Recursion is real** — treats automation as one-pass, not multi-step

### Why Human Validation Still Required

Even with context, the AI code needs review for:
- **Does the recursion guard work?** (static variables reset per transaction; this one is safe)
- **Does the guard break legitimate re-entry?** (No; it only prevents same-trigger re-entry in same transaction)
- **Is error handling appropriate?** (Yes; logging in try-finally prevents silent failures)
- **Is the business logic correct?** (AI can't infer "should we include deleted opps?" or "what if amount is null?")
- **Does it handle all flows?** (Need to verify the Flow doesn't bypass this, call an API, etc.)

### The Real Difference

**Without context**: AI fixes the obvious symptom (SOQL loop) and stops.

**With context**: AI understands the hidden system (Flow interaction, bulk expectations, multi-tenant constraints) and fixes the root cause (architecture, not just query).

**With validation**: Human catches edge cases AI can't (null amounts, deleted records, permission gaps, flow side effects).

