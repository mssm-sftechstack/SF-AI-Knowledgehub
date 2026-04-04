# Salesforce Development Rules for GitHub Copilot

This repository contains a Salesforce DX project. Apply all rules below to every suggestion, completion, and generated file.

---

## Multi-Tenant Governor Limits

Salesforce runs on a shared infrastructure where every org shares the same server resources. The platform enforces hard limits per transaction. The most critical: 100 SOQL queries per transaction, 150 DML statements, 6MB heap, 10 seconds CPU time.

**Never put SOQL or DML inside a for loop.** This is the single most common way to hit governor limits. Instead, collect all the IDs you need, run one query outside the loop, put the results in a `Map<Id, SObject>`, and look up records from the map inside the loop.

Bad:
```apex
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
}
```

Good:
```apex
Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();
Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}
```

**Always bulkify.** Every Apex method must handle 200+ records in one call. Salesforce processes up to 200 records per trigger execution. Design for collections, not single records.

**Never hardcode IDs.** Record Type IDs, Queue IDs, and Profile IDs change between orgs. Resolve them dynamically:

```apex
Id rtId = Schema.SObjectType.Opportunity
    .getRecordTypeInfosByDeveloperName()
    .get('Enterprise')
    .getRecordTypeId();
```

---

## Security and Access Control

**Field-level security in SOQL.** Every SOQL query must use `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()`. This enforces that the running user can actually read the fields being queried.

```apex
// WITH SECURITY_ENFORCED — throws exception if field is inaccessible
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WHERE Industry = 'Technology'
    WITH SECURITY_ENFORCED
];

// Security.stripInaccessible — removes inaccessible fields silently
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Name, AnnualRevenue FROM Account WHERE Industry = 'Technology']
);
List<Account> accounts = (List<Account>) decision.getRecords();
```

**Sharing model.** All Apex classes must declare `with sharing`. This respects the org's record-level sharing rules. If a class legitimately needs to bypass sharing (e.g. a Selector running cross-user queries), use `inherited sharing` and document the reason in a comment.

**Permission Sets over Profiles.** Never add new field or object permissions to Profiles. Always use Permission Sets. This keeps Profiles stable across orgs and makes permission assignment reversible.

**No callouts in triggers.** HTTP callouts are not allowed in synchronous trigger context. Use a Queueable class that implements `Database.AllowsCallouts` and enqueue it from the trigger handler.

---

## Architecture Pattern

All Apex in this project follows a four-layer structure:

**Trigger** — zero logic, one line delegating to the handler:
```apex
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    new AccountTriggerHandler().run();
}
```

**TriggerHandler** — routes by context (before/after, insert/update), no SOQL, no business logic.

**Service** — all business logic, no SOQL. Receives collections of records and IDs, calls the Selector for data.

**Selector** — all SOQL lives here. Uses `inherited sharing` so the caller's sharing context applies.

Never create a second trigger on an object that already has one. Extend the handler instead.

---

## Testing Requirements

Every new Apex class requires a paired test class. Test classes must meet all of these:

- `@IsTest(SeeAllData=false)` on every test class — never `seeAllData=true`
- 90%+ code coverage minimum
- At least one bulk test method that processes 200 records
- `System.runAs()` wrapping any test that calls code using `Security.stripInaccessible()` or `WITH USER_MODE` — System Admin skips FLS silently, which causes false passes
- `System.assertEquals(expected, actual, 'message')` on specific field values, not just record counts

---

## Deployment Rules

Deploy metadata in this order when multiple types are involved:

1. Custom Objects and Fields
2. Custom Metadata Types
3. Custom Settings
4. Custom Tabs (must be in org before Permission Sets reference them)
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

Always validate before deploying. Never deploy without user confirmation.

---

## Code Style

- No AI-pattern comments: never write "This ensures that...", "It's important to note...", "is responsible for..."
- No em dashes in comments
- Class names: PascalCase. Methods: camelCase. Test classes: ClassNameTest. Selectors: ObjectNameSelector
- API version 62.0 in all metadata files
- `sf` CLI v2 only — never `sfdx` commands
