# Salesforce Apex Skill

**Apex class architecture, trigger handlers, service layers, async patterns, governor limits, and security enforcement.**

## Architecture — Always Follow This Layered Pattern

```
Trigger (thin — delegation only)
  └── TriggerHandler  (context routing — no SOQL, no DML)
        └── Service   (business logic — calls Selector)
              └── Selector (ALL SOQL lives here — inherited sharing)
```

Never skip layers. Never put logic directly in a trigger.

## Sharing Keywords — When to Use Each

```apex
// with sharing — user-facing service classes, AuraEnabled methods
// respects record-level access (role hierarchy, sharing rules, OWD)
public with sharing class AccountService { }

// without sharing — documented exceptions only (system integrations, admin ops)
// MUST have a comment explaining why
public without sharing class IntegrationService {
    // runs as system for scheduled sync job — no user context available
    // Approved: [architect name, date]
}

// inherited sharing — selectors and utility classes called from multiple places
// takes the sharing context of the calling class — safest default for reusable code
public inherited sharing class AccountSelector { }
```

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_keywords_sharing.htm

---

## Salesforce Order of Execution — Know This Before Writing Any Trigger or Flow

Every DML operation fires in this exact sequence. Misunderstanding this is the #1 source of trigger-vs-flow interaction bugs.

1.  System validation (required fields, unique fields, field formats)
2.  **Before-save record-triggered Flows** — fires BEFORE before-triggers
3.  **Before Apex Triggers**
4.  System validation rules (format, uniqueness)
5.  Custom validation rules
6.  Duplicate rules
7.  Record saved to database (not a full transaction commit yet)
8.  **After Apex Triggers** — fires BEFORE after-save flows
9.  Assignment rules
10. Auto-response rules
11. Workflow rules (field updates restart the sequence from step 1 on the same record)
12. Escalation rules
13. **After-save record-triggered Flows** — fires AFTER after triggers, AFTER workflow
14. Entitlement rules
15. Roll-up summary and cross-object formula recalculation
16. Criteria-based sharing evaluation
17. Full transaction commits to the database

**Critical implications:**
- Before-save Flows fire BEFORE before triggers — decide which layer owns each piece of logic
- **After-save Flows fire AFTER after triggers (step 13 vs step 8)** — after-trigger logic is fully complete when after-save flows run
- Workflow field updates restart execution from step 1 — guard against infinite loops with a flag field
- Flows and triggers share the same governor limit budget per transaction
- Platform Events published inside a transaction survive a rollback — they are NOT rolled back
- Criteria-based sharing (step 16) runs before commit — manual share records created in triggers are evaluated here

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm

---

## Step-by-Step: Creating New Apex

1. Check for an existing trigger on the object — extend the handler, never create a second trigger.
2. Create files in order: Selector → Service → TriggerHandler → Trigger → Test class.
3. Run `/sf-security` check before finalising.
4. Generate test class with `/sf-test` (90%+ coverage required).
5. Confirm with user before deploying.

---

## Selector Class Template (ALL SOQL lives here)

```apex
public inherited sharing class AccountSelector {

    // all queries go through here — no SOQL in service or handler classes
    public List<Account> getByIds(Set<Id> ids) {
        // LIMIT 2000 guards against the 50,000-row SOQL governor limit
        // for result sets larger than 2,000 use Database.QueryLocator in a Batch job
        return [
            SELECT Id, Name, BillingCountry, OwnerId
            FROM   Account
            WHERE  Id IN :ids
            WITH   SECURITY_ENFORCED
            LIMIT  2000
        ];
    }

    public List<Account> getActiveByOwnerId(Id ownerId) {
        return [
            SELECT Id, Name, Status__c
            FROM   Account
            WHERE  OwnerId = :ownerId
            AND    Status__c = 'Active'
            WITH   SECURITY_ENFORCED
            LIMIT  1000
        ];
    }
}
```

---

## Service Class Template

```apex
public with sharing class AccountService {

    private static final AccountSelector selector = new AccountSelector();

    public static void processAccounts(Set<Id> accountIds) {
        if (accountIds == null || accountIds.isEmpty()) {
            throw new IllegalArgumentException('accountIds cannot be null or empty');
        }

        List<Account> accounts = selector.getByIds(accountIds);
        List<Account> toUpdate = new List<Account>();

        for (Account acc : accounts) {
            acc.Status__c = 'Processed';
            toUpdate.add(acc);
        }

        if (!toUpdate.isEmpty()) {
            if (!Schema.sObjectType.Account.isUpdateable()) {
                throw new AuraHandledException('No permission to update Accounts.');
            }
            update toUpdate;
        }
    }
}
```

---

## Trigger Framework

### Trigger — Delegates Only
```apex
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    AccountTriggerHandler handler = new AccountTriggerHandler();
    if (Trigger.isBefore) {
        if (Trigger.isInsert) handler.onBeforeInsert(Trigger.new);
        if (Trigger.isUpdate) handler.onBeforeUpdate(Trigger.new, Trigger.oldMap);
        if (Trigger.isDelete) handler.onBeforeDelete(Trigger.old);
    }
    if (Trigger.isAfter) {
        if (Trigger.isInsert)   handler.onAfterInsert(Trigger.new);
        if (Trigger.isUpdate)   handler.onAfterUpdate(Trigger.new, Trigger.oldMap);
        if (Trigger.isDelete)   handler.onAfterDelete(Trigger.old, Trigger.oldMap);
        if (Trigger.isUndelete) handler.onAfterUndelete(Trigger.new);
    }
}
```

### TriggerHandler — Routes Context, No Logic
```apex
public with sharing class AccountTriggerHandler {

    public void onBeforeInsert(List<Account> newRecords) {
        if (TriggerBypass.isActive('Account')) return;
        AccountService.setDefaults(newRecords);
    }

    public void onAfterInsert(List<Account> newRecords) {
        if (TriggerBypass.isActive('Account')) return;
        AccountService.sendWelcomeNotifications(newRecords);
    }

    public void onBeforeUpdate(List<Account> newRecords, Map<Id, Account> oldMap) {
        if (TriggerBypass.isActive('Account')) return;
        AccountService.validateChanges(newRecords, oldMap);
    }
}
```

### Bypass — Custom Permission Based (Preferred over Static Boolean)
```apex
public class TriggerBypass {
    // checks Custom Permission — configurable per user/Permission Set without code change
    public static Boolean isActive(String objectName) {
        return FeatureManagement.checkPermission('Bypass_' + objectName + '_Trigger');
    }
}
```
Custom Permission name pattern: `Bypass_Account_Trigger`, `Bypass_Opportunity_Trigger`, etc.
Assign via Permission Set for data migrations or integrations — no code deployment needed.

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_custom_permissions.htm

---

## Bulkification — No Exceptions

```apex
// WRONG — SOQL in loop, fails at 101 records
for (Opportunity opp : opportunities) {
    Account acc = [SELECT Id FROM Account WHERE Id = :opp.AccountId];
}

// CORRECT — single query, O(1) Map lookup
Set<Id> accountIds = new Set<Id>();
for (Opportunity opp : opportunities) {
    accountIds.add(opp.AccountId);
}
Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :accountIds WITH SECURITY_ENFORCED]
);
for (Opportunity opp : opportunities) {
    Account acc = accountMap.get(opp.AccountId);
    if (acc != null) { /* process */ }
}
```

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_bestpract.htm

---

## Error Handling

```apex
// AuraHandledException for LWC-facing errors — must call setMessage() explicitly
private static AuraHandledException auraErr(String msg) {
    AuraHandledException e = new AuraHandledException(msg);
    e.setMessage(msg);
    return e;
}

// DML error handling — collect errors, don't throw on first failure
List<Database.SaveResult> results = Database.insert(records, false);
List<String> errors = new List<String>();
for (Database.SaveResult r : results) {
    if (!r.isSuccess()) {
        for (Database.Error err : r.getErrors()) {
            errors.add(err.getMessage());
        }
    }
}
if (!errors.isEmpty()) {
    throw auraErr('Insert failed: ' + String.join(errors, '; '));
}
```

## Mixed DML Exception — Setup vs Non-Setup Objects

You cannot insert/update/delete setup objects (User, PermissionSetAssignment, Group, GroupMember) and non-setup objects (Account, Contact, custom objects) in the same transaction.

```apex
// VIOLATION — throws System.DmlException: MIXED_DML_OPERATION
insert new Account(Name = 'Test');
insert new User(...);   // setup object — different transaction required

// FIX — use @future to execute setup DML in a separate transaction
@future
public static void assignPermissionSet(Id userId, Id permSetId) {
    insert new PermissionSetAssignment(
        AssigneeId      = userId,
        PermissionSetId = permSetId
    );
}
```

This is the most common test class trap — creating Users and non-setup records in the same `@TestSetup` or test method throws Mixed DML. Create Users in a separate `@future` call or isolate them before `System.runAs()`.

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_dml_non_mix_sobjects.htm

---

## Async Apex — When to Use What

| Scenario | Pattern |
|---|---|
| Simple async, no object params | `@future(callout=true)` |
| Async with complex params, chaining | `Queueable + Database.AllowsCallouts` |
| Large data volume (>200 records) | `Database.Batchable` (batch size 200) |
| Time-based scheduling | `Schedulable` |
| HTTP callout from trigger context | `Queueable + Database.AllowsCallouts` |
| Decoupled cross-system event | Platform Events (see sf-platform-events) |
| Guaranteed cleanup after failure | `Queueable + System.Finalizer` (API 53.0+) |

```apex
// Queueable with callout — correct pattern for trigger-initiated HTTP calls
// bulk-fetch all records in one pass — never call a service per record inside execute()
public class WeatherRefreshJob implements Queueable, Database.AllowsCallouts {
    private List<Id> recordIds;

    public WeatherRefreshJob(List<Id> ids) {
        this.recordIds = ids;
    }

    public void execute(QueueableContext ctx) {
        System.attachFinalizer(new WeatherRefreshFinalizer());
        // one bulk operation — not one HTTP call per record
        WeatherService.fetchAndSaveBulk(recordIds);
    }
}
System.enqueueJob(new WeatherRefreshJob(newRecordIds));
```

### Transaction Finalizer — Guaranteed Cleanup After Failure (API 53.0+)

```apex
// runs after the Queueable completes — even on unhandled exception
public class WeatherRefreshFinalizer implements System.Finalizer {
    public void execute(FinalizerContext ctx) {
        if (ctx.getResult() == ParentJobResult.UNHANDLED_EXCEPTION) {
            // log failure, publish a Platform Event to trigger retry — do NOT throw from here
            System.debug(LoggingLevel.ERROR,
                'WeatherRefreshJob failed: ' + ctx.getException().getMessage());
        }
    }
}
// attach at the top of Queueable.execute() before any processing:
// System.attachFinalizer(new WeatherRefreshFinalizer());
```

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_transaction_finalizers.htm

### Batch Apex — For Large Data Volumes
```apex
public class AccountCleanupBatch implements Database.Batchable<SObject>, Database.Stateful {

    private Integer processed = 0;

    public Database.QueryLocator start(Database.BatchableContext ctx) {
        // QueryLocator scales to 50M rows; for large volumes always use this over List<SObject>
        return Database.getQueryLocator(
            'SELECT Id, Status__c FROM Account WHERE Status__c = \'Stale\''
        );
    }

    public void execute(Database.BatchableContext ctx, List<Account> scope) {
        for (Account acc : scope) {
            acc.Status__c = 'Archived';
            processed++;
        }
        update scope; // single DML per batch chunk
    }

    public void finish(Database.BatchableContext ctx) {
        AsyncApexJob job = [
            SELECT Id, Status, NumberOfErrors
            FROM   AsyncApexJob
            WHERE  Id = :ctx.getJobId()
        ];
        if (job.NumberOfErrors > 0) {
            // publish a Platform Event or send notification — not System.debug
        }
    }
}
Database.executeBatch(new AccountCleanupBatch(), 200); // 200 is the optimal batch size
```

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm

---

## Apex Security Enforcement Quick Reference

| Action | Enforcement Method |
|---|---|
| Read — throw if field inaccessible | `WITH SECURITY_ENFORCED` in SOQL |
| Read — strip inaccessible fields | `Security.stripInaccessible(AccessType.READABLE, records)` |
| Insert | `Schema.sObjectType.Obj.isCreateable()` |
| Update | `Schema.sObjectType.Obj.isUpdateable()` |
| Delete | `Schema.sObjectType.Obj.isDeletable()` |
| Query access | `Schema.sObjectType.Obj.isAccessible()` |
| Record-level sharing | `with sharing` on class declaration |

Custom Metadata Types (`__mdt`) and Hierarchy Custom Settings are exempt from FLS — no `WITH SECURITY_ENFORCED` needed on those queries.

Always run `/sf-security` before finalising any Apex class.

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_security_sharing_chapter.htm

---

## Governor Limits Reference

| Limit | Per Transaction | Design Target |
|---|---|---|
| SOQL queries | 100 | < 20 |
| SOQL rows returned | 50,000 | < 10,000 |
| DML statements | 150 | < 30 |
| DML rows | 10,000 | < 5,000 |
| CPU time | 10,000ms | < 3,000ms |
| Heap size | 6MB | < 2MB |
| Callouts | 100 | < 10 |

Use `Limits.getQueries()`, `Limits.getDmlStatements()` to check remaining at runtime during debugging (do not ship these checks in production code).

Official source: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm

---

## Naming Conventions

| Type | Pattern | Example |
|---|---|---|
| Class | PascalCase | `AccountTriggerHandler` |
| Test class | `ClassNameTest` | `AccountTriggerHandlerTest` |
| Trigger | `ObjectNameTrigger` | `AccountTrigger` |
| Service | `ObjectNameService` | `AccountService` |
| Selector | `ObjectNameSelector` | `AccountSelector` |
| Batch | `ObjectNameBatch` | `AccountSyncBatch` |
| Queueable | `DescriptiveNameJob` | `WeatherRefreshJob` |
| Interface | `IDescriptiveName` | `ICalloutService` |
| Custom Exception | `DescriptiveException` | `WeatherApiException` |

---

## Deployment (Confirm with User First)
```bash
sf project deploy start --metadata ApexClass --target-org <your-org-alias>
sf apex run test --class-names AccountServiceTest --target-org <your-org-alias> --wait 10
sf apex run test --test-level RunLocalTests --code-coverage --target-org <your-org-alias> --wait 30
```

---

## Apex Quality Scoring (Ship only at 120+/150)

Score every Apex class before marking it ready. Minimum threshold: 120/150 (80%).

| Dimension | Points | Failing Condition |
|---|---|---|
| Bulkification | 25 | SOQL or DML inside any loop; no Set/Map for ID collections |
| Security | 25 | Missing WITH SECURITY_ENFORCED or Security.stripInaccessible(); no sharing keyword; hardcoded ID |
| Testing | 25 | Coverage below 90%; no bulk test (200 records); no negative test; no assert on field values |
| Architecture | 20 | Logic in trigger body; no selector layer; service class has SOQL; no layer separation |
| Error Handling | 15 | Empty catch block; swallowed exception; AuraHandledException without setMessage() |
| Naming | 15 | Class not PascalCase; test class not [Name]Test; selector not [Object]Selector; method not camelCase |
| Performance | 15 | No LIMIT on queries that could return >1000 rows; synchronous callout in trigger context |
| Code Quality | 10 | AI-pattern comments; em dashes in comments; dead code left in; commented-out blocks shipped |
| **TOTAL** | **150** | |

**Score 120-150** → ready to ship
**Score 100-119** → fix failing dimensions first
**Score below 100** → reject, do not deploy

## Official Sources
- [Apex Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_intro.htm)
- [Apex Triggers Best Practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_bestpract.htm)
- [Apex Order of Execution](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)
- [Apex Security and Sharing](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_security_sharing_chapter.htm)
- [Apex Mixed DML Exception](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_dml_non_mix_sobjects.htm)
- [Apex Transaction Finalizers](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_transaction_finalizers.htm)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [Asynchronous Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_async_overview.htm)
- [Batch Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm)
- [Queueable Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_queueing_jobs.htm)
- [WITH SECURITY_ENFORCED](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_with_security_enforced.htm)
- [Custom Permissions](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_custom_permissions.htm)
- [Well-Architected: Reliable](https://architect.salesforce.com/well-architected/trusted/reliable)
- [Well-Architected: Composable](https://architect.salesforce.com/well-architected/adaptable/composable)
- [Salesforce Security Implementation Guide](https://help.salesforce.com/s/articleView?id=sf.security_overview.htm)
