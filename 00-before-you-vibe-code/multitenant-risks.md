# Multi-Tenant Risks Without Guardrails

**What goes wrong when AI tools generate Salesforce code without knowing the multi-tenant rules.**

---

## What multi-tenancy means

Salesforce hosts over 150,000 customer orgs on shared servers. Your code runs in the same JVM heap as code from thousands of other customers. Salesforce enforces **governor limits** on every transaction to prevent any single org from consuming resources that would slow down everyone else.

Governor limits aren't optional. They can't be increased beyond hard ceilings (only a few have soft limits you can request via a case). They apply to every Apex transaction: triggers, batch jobs, future methods, queueable jobs, and anonymous Apex.

The AI tools generating your code were trained on billions of lines of public code. Almost none of that code was Salesforce Apex written at enterprise scale. The AI doesn't know these limits unless you explicitly provide them in a context file.

---

## The 5 most common AI-generated mistakes

| Mistake | What the AI writes | What actually happens at scale | The fix |
|---|---|---|---|
| SOQL in a loop | `for(opp : opps) { Account a = [SELECT Id FROM Account WHERE Id = :opp.AccountId]; }` | Hits the 101-query limit at 101 records. Every record in the batch fails. | Move the query outside the loop. Use a `Map<Id, Account>` keyed by ID. |
| Hardcoded Record Type ID | `opp.RecordTypeId = '012000000000001AAA';` | Works in the dev org where you copied the ID. Breaks in every sandbox, scratch org, and other customer org because Record Type IDs are org-specific. | Use `SObjectType.Opportunity.getRecordTypeInfosByDeveloperName().get('MyType').getRecordTypeId()` |
| Missing sharing keyword | `public class AccountService { ... }` | Runs in system context, ignoring the org's record sharing rules. Users can read, edit, or delete records they should never see. | Declare `public with sharing class AccountService` on every class unless you have an explicit documented reason to use `without sharing`. |
| No FLS enforcement | `[SELECT Salary__c FROM Contact WHERE Id = :contactId]` | Returns the field value even if the running user's profile has no read access to `Salary__c`. Silent data exposure. Fails AppExchange security review. | Add `WITH SECURITY_ENFORCED` to the SOQL, or call `Security.stripInaccessible(AccessType.READABLE, records)` before returning results. |
| Second trigger on same object | Creating `AccountTrigger2.trigger` because `AccountTrigger.trigger` already exists | Both triggers fire in undefined order. Logic conflicts. The platform allows it but it causes unpredictable behavior and makes the codebase unmaintainable. | One trigger per object. Add your logic to the existing trigger handler class. |

---

## **Anti-Pattern**: SOQL in a loop

```apex
// **Anti-Pattern**
List<Opportunity> opps = [SELECT Id, AccountId FROM Opportunity WHERE StageName = 'Prospecting'];
for (Opportunity opp : opps) {
    // This fires a new SOQL query for every iteration
    Account acc = [SELECT Name, Industry FROM Account WHERE Id = :opp.AccountId];
    System.debug(acc.Name);
}
// Result: 101st record throws LimitException
```

```apex
// Correct pattern
List<Opportunity> opps = [SELECT Id, AccountId FROM Opportunity WHERE StageName = 'Prospecting'];

Set<Id> accountIds = new Set<Id>();
for (Opportunity opp : opps) {
    accountIds.add(opp.AccountId);
}

Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Name, Industry FROM Account WHERE Id IN :accountIds]
);

for (Opportunity opp : opps) {
    Account acc = accountMap.get(opp.AccountId);
    System.debug(acc.Name);
}
// Result: 2 queries total, regardless of how many Opportunities
```

---

## Governor limit quick reference

Design targets are conservative ceilings for well-behaved code. Hitting the hard limit in production means an unhandled exception and a failed transaction.

| Limit | Per transaction (hard) | Design target |
|---|---|---|
| SOQL queries | 100 | fewer than 20 |
| SOQL rows returned | 50,000 | fewer than 10,000 |
| DML statements | 150 | fewer than 30 |
| DML rows | 10,000 | fewer than 5,000 |
| CPU time | 10,000ms | fewer than 3,000ms |
| Heap size | 6MB | fewer than 2MB |
| Callouts | 100 | fewer than 10 |
| Future method calls | 50 | fewer than 10 |
| Queueable jobs enqueued | 50 | fewer than 10 |
| Batch execute() chunks | unlimited | batch size 200 |

Limits reset at the start of each transaction. A batch job's `execute()` method gets a fresh set of limits per chunk.

---

## Why "it works in dev" is dangerous

Developer Edition orgs ship with a handful of sample records. A SOQL-in-loop bug against 5 Account records executes 5 queries, well under the 100-query limit. The bug is invisible.

In production, the same trigger fires against a bulk import of 300 records. The first 100 iterations use up all 100 query slots. Record 101 throws `System.LimitException: Too many SOQL queries: 101`. The entire batch of 300 records fails and rolls back.

**Always test with 200+ records.** Salesforce processes triggers in batches of up to 200 records at a time. Any code that works correctly at 200 records will work correctly at any volume. Any code that breaks at 200 records will break in production.

The standard test pattern:

```apex
@IsTest
static void testBulkInsert() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
        accounts.add(new Account(Name = 'Test Account ' + i));
    }
    Test.startTest();
    insert accounts;
    Test.stopTest();

    // assert results for all 200 records
    List<Account> result = [SELECT Id, Name FROM Account WHERE Name LIKE 'Test Account%'];
    System.assertEquals(200, result.size(), 'All 200 accounts should be inserted');
}
```

---

## How to tell the AI about these rules

Add a context file to your project root that lists these constraints explicitly. The AI reads this file before every response and applies the rules automatically.

Minimum rules to include in any Salesforce context file:

- Never write SOQL inside a for loop
- Always use `with sharing` on every Apex class
- Always add `WITH SECURITY_ENFORCED` to SOQL queries
- Never hardcode Record Type IDs, org IDs, or credentials
- One trigger per object -- extend the handler, never create a second trigger
- Design for 200+ records in every trigger and batch
- Always write a test class with 90%+ coverage for every new Apex class

See [02-ai-tool-setup/](../02-ai-tool-setup/) for complete ready-to-use context file templates.
