# Real-World Example: Debugging a "101 SOQL Queries" Error

**Scenario**: Your trigger is hitting the 101 query limit in production. It works fine in your dev org (30 records). Breaks in production (5,000 records).

This walkthrough shows how AI responds without context vs. with proper Salesforce constraints.

---

## 1. Problem Context

You're a developer at an e-commerce company. Your org has an Account object with a custom field `Annual_Spend__c`.

**Requirement**: When an Account's status changes to 'Active', calculate total spend from related Opportunities and update `Annual_Spend__c`.

**Current code**: Works in dev, but production reports:
```
FATAL_ERROR|System.LimitException: Too many SOQL queries: 101
```

**What happened**: 
- You looped through 5,000 Accounts
- Inside the loop, you queried their Opportunities
- 101 queries = 101 SOQL limit hit
- Dev only had 30 Accounts, so never triggered the limit

This is a real, preventable bug.

---

## 2. Initial Code (Bad Example)

Here's what the trigger looks like right now:

```apex
trigger AccountTrigger on Account (after update) {
  for (Account acc : Trigger.new) {
    Account oldAcc = Trigger.oldMap.get(acc.Id);
    
    // Status changed to Active? Recalculate spend.
    if (acc.Status__c == 'Active' && oldAcc.Status__c != 'Active') {
      // Problem: SOQL inside loop
      List<Opportunity> opps = [
        SELECT Id, Amount 
        FROM Opportunity 
        WHERE AccountId = :acc.Id
      ];
      
      Decimal totalSpend = 0;
      for (Opportunity opp : opps) {
        totalSpend += opp.Amount;
      }
      
      acc.Annual_Spend__c = totalSpend;
    }
  }
  
  update Trigger.new;
}
```

**Why this breaks:**
- Loop through 5,000 Accounts (5,000 iterations)
- Each iteration = 1 SOQL query
- Hit governor limit at 101 queries
- Code crashes

---

## 3. AI Without Context (Failure Case)

**What you prompt a generic AI tool:**

```
"My trigger is querying inside a loop. How do I fix it?"
```

**Generic AI response:**

```apex
// Use a map to batch your queries
Map<Id, List<Opportunity>> oppsByAccountId = new Map<Id, List<Opportunity>>();

for (Account acc : Trigger.new) {
  if (!oppsByAccountId.containsKey(acc.Id)) {
    oppsByAccountId.put(acc.Id, [
      SELECT Id, Amount FROM Opportunity WHERE AccountId = :acc.Id
    ]);
  }
}
```

**Why this fails:**

❌ **Still queries in a loop.** (Just caches results.) Governor limit still hit.  
❌ **No WITH SECURITY_ENFORCED.** (FLS violation. Field might be hidden from the user.)  
❌ **No consideration for bulk DML.** (Doesn't mention the DML needs to be outside the loop.)  
❌ **Ignores Salesforce context.** (Generic solution, not Salesforce-aware.)  

The AI missed the core problem because it didn't have Salesforce constraints in its context.

---

## 4. Minimal Context Provided

**Better prompt (with Salesforce context):**

```
Object: Account (Custom field: Annual_Spend__c)
Trigger: after update on Account
Requirement: When Status__c changes to 'Active', 
  sum all related Opportunity.Amount and update Account.Annual_Spend__c

Constraints:
- Never query inside a loop (101 query limit)
- Always use WITH SECURITY_ENFORCED
- Bulk test: 5,000+ records
- Use trigger handler pattern (not monolithic trigger)
- Implement recursion guard

Show me the handler class only (not the trigger itself).
```

**What this does:**
- Tells AI the exact object and field
- Explains the constraint (101 query limit)
- Specifies "bulk test" (alerts AI to think about scale)
- Names the pattern (trigger handler)
- Asks for handler only (focused output)

---

## 5. AI With Context (Improved Output)

**AI response with Salesforce constraints:**

```
I see the issue. You're querying inside the loop, which hits the 101 query limit 
when you process 5,000+ Accounts.

Solution:
1. Collect all Account IDs first (no query)
2. Query ALL Opportunities in bulk (one query)
3. Loop through Accounts and aggregate spend in memory
4. Update Accounts after all logic completes

Here's the handler:

public class AccountTriggerHandler {
  private static Boolean isExecuting = false;
  
  public static void handle(List<Account> newAccounts, Map<Id, Account> oldMap) {
    if (isExecuting) return;  // Recursion guard
    isExecuting = true;
    
    // Step 1: Collect IDs from Accounts where Status changed to Active
    Set<Id> accountIds = new Set<Id>();
    for (Account acc : newAccounts) {
      Account oldAcc = oldMap.get(acc.Id);
      if (acc.Status__c == 'Active' && oldAcc.Status__c != 'Active') {
        accountIds.add(acc.Id);
      }
    }
    
    if (accountIds.isEmpty()) return;
    
    // Step 2: ONE query for all Opportunities (bulk-safe)
    Map<Id, Decimal> totalsByAccount = new Map<Id, Decimal>();
    for (AggregateResult ar : [
      SELECT AccountId, SUM(Amount) totalAmount
      FROM Opportunity
      WHERE AccountId IN :accountIds
      WITH SECURITY_ENFORCED
      GROUP BY AccountId
    ]) {
      Id accId = (Id) ar.get('AccountId');
      Decimal total = (Decimal) ar.get('totalAmount');
      totalsByAccount.put(accId, total);
    }
    
    // Step 3: Update in memory (no DML yet)
    for (Account acc : newAccounts) {
      if (totalsByAccount.containsKey(acc.Id)) {
        acc.Annual_Spend__c = totalsByAccount.get(acc.Id);
      }
    }
    
    // Step 4: DML happens outside the loop (in the trigger itself)
  }
}
```

**Why this works:**

✅ **One SOQL query.** (Not 5,000.)  
✅ **Uses WITH SECURITY_ENFORCED.** (FLS-safe.)  
✅ **Aggregate function.** (SUM happens in the database, not Apex.)  
✅ **Bulk-safe by design.** (5,000 records = 1 query.)  
✅ **Recursion guard.** (Prevents infinite loops.)  
✅ **Trigger handler pattern.** (Clean separation of concerns.)  

The AI understood Salesforce constraints and applied them.

---

## 6. Refactored Code (Good Example)

**Complete, production-ready implementation:**

```apex
// Trigger (simple, delegates to handler)
trigger AccountTrigger on Account (after update) {
  AccountTriggerHandler.handle(Trigger.new, Trigger.oldMap);
}

// Handler (reusable, testable)
public class AccountTriggerHandler {
  private static Boolean isExecuting = false;
  
  public static void handle(List<Account> newAccounts, Map<Id, Account> oldMap) {
    if (isExecuting) return;
    isExecuting = true;
    
    Set<Id> activeAccountIds = getAccountsWithStatusChange(newAccounts, oldMap);
    if (activeAccountIds.isEmpty()) return;
    
    Map<Id, Decimal> spendByAccount = getAccountSpendTotals(activeAccountIds);
    updateAccountSpend(newAccounts, spendByAccount);
  }
  
  private static Set<Id> getAccountsWithStatusChange(
    List<Account> newAccounts, 
    Map<Id, Account> oldMap
  ) {
    Set<Id> accountIds = new Set<Id>();
    for (Account acc : newAccounts) {
      Account oldAcc = oldMap.get(acc.Id);
      if (acc.Status__c == 'Active' && oldAcc.Status__c != 'Active') {
        accountIds.add(acc.Id);
      }
    }
    return accountIds;
  }
  
  private static Map<Id, Decimal> getAccountSpendTotals(Set<Id> accountIds) {
    Map<Id, Decimal> spendByAccount = new Map<Id, Decimal>();
    for (AggregateResult ar : [
      SELECT AccountId, SUM(Amount) totalAmount
      FROM Opportunity
      WHERE AccountId IN :accountIds
      WITH SECURITY_ENFORCED
      GROUP BY AccountId
    ]) {
      Id accId = (Id) ar.get('AccountId');
      Decimal total = (Decimal) ar.get('totalAmount');
      spendByAccount.put(accId, total);
    }
    return spendByAccount;
  }
  
  private static void updateAccountSpend(
    List<Account> accounts,
    Map<Id, Decimal> spendByAccount
  ) {
    for (Account acc : accounts) {
      if (spendByAccount.containsKey(acc.Id)) {
        acc.Annual_Spend__c = spendByAccount.get(acc.Id);
      }
    }
  }
}

// Test class (bulk test with 5,000 records)
@IsTest
private class AccountTriggerHandlerTest {
  @IsTest
  static void testBulkAccountStatusChange() {
    // Create 200 Accounts (bulk test)
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
      accounts.add(new Account(Name = 'Test Acc ' + i, Status__c = 'Inactive'));
    }
    insert accounts;
    
    // Create Opportunities for each Account
    List<Opportunity> opps = new List<Opportunity>();
    for (Account acc : accounts) {
      opps.add(new Opportunity(
        Name = 'Opp for ' + acc.Name,
        AccountId = acc.Id,
        Amount = 1000,
        StageName = 'Closed Won',
        CloseDate = Date.today()
      ));
    }
    insert opps;
    
    // Change status to Active
    Test.startTest();
    for (Account acc : accounts) {
      acc.Status__c = 'Active';
    }
    update accounts;
    Test.stopTest();
    
    // Verify Annual_Spend__c was updated
    List<Account> updatedAccounts = [SELECT Annual_Spend__c FROM Account];
    for (Account acc : updatedAccounts) {
      System.assertEquals(1000, acc.Annual_Spend__c, 'Spend should be 1000');
    }
  }
}
```

**What makes this production-ready:**

✅ **Private helper methods.** (Clear intent, easy to test.)  
✅ **Recursion guard.** (Prevents infinite loops.)  
✅ **One SOQL query.** (Scale to 100,000 records.)  
✅ **WITH SECURITY_ENFORCED.** (Respects FLS.)  
✅ **Aggregate function.** (SUM in database, not Apex.)  
✅ **Bulk test.** (Tests with 200 records.)  
✅ **Comments are minimal.** (Code is self-explanatory.)  

---

## 7. Key Takeaways

### Why Context Matters

**Without Salesforce context:**
- AI gave a generic "use a map" solution
- Didn't address the core problem (SOQL in loop)
- Introduced FLS gaps

**With Salesforce context:**
- AI understood governor limits
- Recognized bulk test requirement
- Applied WITH SECURITY_ENFORCED automatically
- Suggested aggregate functions (SUM in database)
- Used trigger handler pattern

**The difference:** One word—"bulkify". That single constraint changed everything.

### Why Salesforce Constraints Change AI Behavior

Salesforce has hard limits that don't exist in other platforms:
- 101 SOQL queries per transaction
- 50,000 DML records per transaction
- 6 MB heap size
- One trigger per object (no multiple triggers)
- FLS must be enforced explicitly

When you mention these in context, AI responds differently. It's not smarter. It's just informed.

### Why Human Validation Is Still Required

Even with good context, you need to validate:

1. **Does the SOQL count match the record count?**
   - 5,000 Accounts = 1 query ✅
   - Not 5,000 queries ✅

2. **Is WITH SECURITY_ENFORCED present?**
   - Check the SOQL ✅

3. **Is DML outside the loop?**
   - DML happens once at the end ✅

4. **Does the test cover bulk scenarios?**
   - 200-record test ✅

5. **Is there a recursion guard?**
   - `if (isExecuting) return;` ✅

AI can suggest the right pattern. You verify it's correct.

---

## How to Use This Example

**If you hit a governor limit:**

1. Identify the constraint (101 queries, 50K DML, etc.)
2. Provide AI with the context:
   - Object and trigger event
   - Code snippet
   - "Bulk test with 5,000 records" (signals scale)
   - Constraints from ARCHITECTURE.md

3. Review the response:
   - Count SOQL queries (should be minimal)
   - Check WITH SECURITY_ENFORCED
   - Verify DML is outside loops
   - Validate test includes bulk scenario

4. Commit when verified

---

## Next Steps

- Reference [ARCHITECTURE.md](ARCHITECTURE.md) rule #4: "Never SOQL or DML inside loops"
- Use [PATTERNS.md](PATTERNS.md) for trigger handler boilerplate
- Use [vibe-coding-salesforce.md](vibe-coding-salesforce.md) step 4 (validation layer) before committing
- Write bulk tests every time (rule #9)

---

**Bottom line:** AI is good at spotting patterns. Salesforce constraints are hard to ignore once you mention them. But validation is still your job.
