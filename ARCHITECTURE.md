# Core Architecture and Guardrails

The patterns and rules that keep Salesforce code correct.

---

## The 10 Core Rules

These come from real projects. Most learned the hard way.

1. **No SOQL or DML inside loops.** You'll hit the 101 query limit.
2. **Bulkify for 200+ records.** Dev has 50. Production has 50,000.
3. **Use `with sharing` on every Apex class.** It respects your org's sharing rules.
4. **Use Permission Sets, not Profiles.** Profiles are legacy.
5. **Use `WITH SECURITY_ENFORCED` in every SOQL query.** FLS doesn't enforce itself.
6. **No hardcoded IDs or credentials.** It works in dev. Breaks everywhere else.
7. **One trigger per object.** Extend the handler, never create a second trigger.
8. **No callouts from trigger context.** Use Queueable with Database.AllowsCallouts instead.
9. **90%+ test coverage plus a 200-record bulk test.** Code coverage percentage is misleading without bulk tests.
10. **Validate before you deploy.** It catches surprises.

---

## Layered Apex Architecture

Keep business logic separated from trigger execution.

```
Trigger → Handler → Service → Selector
```

- **Trigger**: One trigger per object. Routes to handler immediately.
- **Handler**: Bulkifies logic. Routes to service for business rules.
- **Service**: Contains business logic. Queries once, uses maps.
- **Selector**: SOQL queries only. WITH SECURITY_ENFORCED. No business logic.

Example:

```apex
// Trigger
trigger Account_Trigger on Account (after update) {
  Account_Handler.handle(Trigger.newMap, Trigger.oldMap);
}

// Handler
public with sharing class Account_Handler {
  public static void handle(Map<Id, Account> newMap, Map<Id, Account> oldMap) {
    List<Account> changed = new List<Account>();
    for (Account newAcc : newMap.values()) {
      if (newAcc.Status__c != oldMap.get(newAcc.Id).Status__c) {
        changed.add(newAcc);
      }
    }
    if (!changed.isEmpty()) {
      Account_Service.processStatusChange(changed);
    }
  }
}

// Service
public with sharing class Account_Service {
  public static void processStatusChange(List<Account> accounts) {
    // Business logic here
  }
}
```

---

## Bulkification Pattern

Collect IDs. Query once. Use map. Iterate.

Wrong:
```apex
for (Account acc : accounts) {
  List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE AccountId = :acc.Id];
  acc.Opp_Count__c = opps.size();
}
```

Right:
```apex
Set<Id> accountIds = new Set<Id>();
for (Account acc : accounts) {
  accountIds.add(acc.Id);
}

Map<Id, List<Opportunity>> oppsByAccount = new Map<Id, List<Opportunity>>();
for (Opportunity opp : [SELECT AccountId FROM Opportunity 
                        WHERE AccountId IN :accountIds
                        WITH SECURITY_ENFORCED]) {
  if (!oppsByAccount.containsKey(opp.AccountId)) {
    oppsByAccount.put(opp.AccountId, new List<Opportunity>());
  }
  oppsByAccount.get(opp.AccountId).add(opp);
}

for (Account acc : accounts) {
  acc.Opp_Count__c = oppsByAccount.containsKey(acc.Id) 
                     ? oppsByAccount.get(acc.Id).size() 
                     : 0;
}
```

---

## SOQL Performance Fundamentals

**Select only fields you use.** Not SELECT *. Pick what you need.

**Filter on indexed fields.** Indexed: Date, CreatedDate, Status, lookups, master-details. Unindexed: custom text fields, rich text, long text.

**Use aggregate for COUNT/SUM.** Not loops.

```apex
// Wrong
List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE AccountId = :accId];
Integer count = opps.size();

// Right
AggregateResult result = [SELECT COUNT(Id) cnt FROM Opportunity 
                          WHERE AccountId = :accId
                          WITH SECURITY_ENFORCED];
Integer count = (Integer)result.get('cnt');
```

**Test at scale.** Dev has 50 records. Prod has 50,000. Test with the real volume.

---

## Security Model

### with sharing

Every class needs it. Respects OWD (org-wide defaults) and sharing rules.

```apex
public with sharing class Account_Service {
  // Respects sharing
}
```

### WITH SECURITY_ENFORCED

Every SOQL query. Enforces field-level security.

```apex
List<Account> accounts = [SELECT Id, Name, Phone FROM Account 
                          WHERE Status = 'Active'
                          WITH SECURITY_ENFORCED];
```

### Field-Level Security

Use Security.stripInaccessible() in tests that enforce FLS.

```apex
List<Account> accounts = [SELECT Id, Name, Phone, Salary__c FROM Account 
                          WITH SECURITY_ENFORCED];
accounts = (List<Account>) Security.stripInaccessible(AccessType.READABLE, accounts);
```

### Permission Sets, Not Profiles

Assign features via Permission Sets. Profiles are legacy.

---

## Async Decision Tree

When to use Queueable, Platform Events, or Scheduled Jobs.

**Use Queueable when:**
- Processing heavy work without blocking the user
- Making callouts (but not from trigger context)
- Chain dependencies (Queueable calls another Queueable)

**Use Platform Events when:**
- Publishing data for subscribers to react to
- Async work that happens after the transaction completes
- Decoupling publisher from subscriber

**Use Scheduled Jobs when:**
- Running on a timer (nightly, hourly)
- Don't need immediate execution
- Need monitoring and retry logic

**Never use from trigger context:**
- No callouts from trigger
- Use Queueable (Database.AllowsCallouts) instead

---

## Order-of-Execution Gotchas

Salesforce executes in this order. Know the surprises.

1. Before trigger fires
2. Before insert/update validations run
3. After trigger fires
4. Assignment rules
5. Auto-response rules
6. Workflow rules (if enabled)
7. Escalation rules

**Gotcha 1: Before-insert dependency**
If your before-insert trigger depends on a field that a before-update flow sets, the flow hasn't run yet. Run the flow after insert instead.

**Gotcha 2: Subflow triggering handler again**
If your after-insert handler calls a flow that updates the same record, the trigger fires again (if it's update-triggered). Use a static flag to prevent recursion.

```apex
public class Account_Handler {
  private static Boolean hasRun = false;
  
  public static void handle(List<Account> records) {
    if (hasRun) return;
    hasRun = true;
    
    // Safe to call flows now
  }
}
```

**Gotcha 3: Rollup summary field triggering parent trigger**
If you update a detail record, its master's rollup summary updates, which triggers the master's trigger. Document this dependency.

---

## Testing Strategy

Write tests that catch real problems.

**Bulk test (200+ records):**
```apex
@IsTest
static void testBulk200Records() {
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
```

**FLS test:**
```apex
@IsTest
static void testFLSEnforced() {
  User standardUser = createStandardUser();
  
  System.runAs(standardUser) {
    List<Account> accounts = Account_Selector.getActiveAccounts();
    // Runs with FLS enforced as standard user, not admin
  }
}
```

**Mock callouts:**
```apex
@IsTest
static void testCallout() {
  Test.setMock(HttpCalloutMock.class, new ExternalSystemMock());
  
  Test.startTest();
  Account_Service.syncAccount(accountId);
  Test.stopTest();
  
  // Assert your service called the mock
}
```

---

## Multi-Tenant Considerations

Salesforce is multi-tenant. Your org shares infrastructure with thousands of others.

- **Don't hardcode IDs.** Use getRecordTypeInfosByDeveloperName() for record types.
- **Don't hardcode field values.** Use picklist metadata, not magic strings.
- **Governor limits are shared.** If your code bloats, it affects everyone on that pod.
- **Query limits are strict.** 101 queries per transaction. No exceptions.
- **Bulk operations hit limits fast.** Test with 200+ records.

---

## Deployment Order

Deploy in this order. Each phase must be independently valid.

1. Custom Objects and Fields
2. Custom Metadata Types
3. Custom Settings
4. Custom Tabs (Permission Sets reference these)
5. Permission Sets and Custom Permissions
6. Apex Classes (non-test)
7. Apex Test Classes
8. Triggers
9. Remote Site Settings and CSP Trusted Sites
10. Named Credentials and External Credentials
11. Flows
12. LWC and Aura Components
13. FlexiPages
14. Page Layouts and Compact Layouts
15. Applications
16. Profiles and Permission Set Assignments

See DEPLOYMENT.md for details.

---

## When to Use Flows vs Apex

**Use Flow when:**
- Automating business processes without code
- Building UI quickly
- Non-developers need to maintain it
- Logic is straightforward and unlikely to change

**Use Apex when:**
- Complex business logic
- Heavy processing or many queries
- Integration with external systems
- Need granular control

**Use both:**
- Flow orchestrates high-level steps
- Apex actions (InvocableApex) handle complex steps
- Flow calls Queueable for async work
