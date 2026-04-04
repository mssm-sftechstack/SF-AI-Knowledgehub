# Async Patterns: Which One to Use

**Four async options, each with a different tradeoff. Pick the wrong one and you'll either hit limits or over-engineer it.**

## Decision Table

| Scenario | Use this | Don't use this |
|---|---|---|
| Simple async, no callout, primitive params | `@future` | Queueable (overkill) |
| Callout from trigger context | `Queueable + Database.AllowsCallouts` | `@future` (can't chain), Batch (too heavy) |
| Large data volume (50k+ records) | `Database.Batchable` | Queueable (no QueryLocator) |
| Time-based scheduling | `Schedulable` | `@future` (can't schedule) |
| Async with chaining or state | `Queueable` | `@future` (no chaining), Batch (awkward to chain) |
| Guaranteed cleanup after failure | `Queueable + System.Finalizer` | Everything else |
| Decoupled cross-system event | `Platform Events + trigger-fired Queueable` | Direct callout from trigger |

## Quick Class Signatures

### @future

```apex
public class AccountUpdater {
    @future
    public static void updateExternalSystem(Set<Id> accountIds) {
        // params must be primitives or collections of primitives — no SObject params
        List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
        // do work
    }
}
```

Use when: no callout, no chaining, no state, primitives only. Simplest option.

### Queueable

```apex
public class ContactSyncJob implements Queueable {
    private List<Id> contactIds;

    public ContactSyncJob(List<Id> contactIds) {
        this.contactIds = contactIds;
    }

    public void execute(QueueableContext ctx) {
        // full Apex limits, can pass SObject params, can chain
    }
}

// Enqueue:
System.enqueueJob(new ContactSyncJob(contactIds));
```

### Database.Batchable

```apex
public class AccountBatchJob implements Database.Batchable<SObject> {

    public Database.QueryLocator start(Database.BatchableContext ctx) {
        return Database.getQueryLocator('SELECT Id, Name FROM Account');
    }

    public void execute(Database.BatchableContext ctx, List<Account> scope) {
        // called once per batch chunk (default 200 records)
    }

    public void finish(Database.BatchableContext ctx) {
        // runs once after all batches complete
    }
}

// Start:
Database.executeBatch(new AccountBatchJob(), 200);
```

Use when: processing 50k+ records. QueryLocator handles pagination automatically.

### Schedulable

```apex
public class DailyCleanupJob implements Schedulable {

    public void execute(SchedulableContext ctx) {
        Database.executeBatch(new AccountBatchJob(), 200);
    }
}

// Schedule via Apex (or Setup > Scheduled Jobs):
String cronExp = '0 0 2 * * ?'; // 2am daily
System.schedule('Daily Cleanup', cronExp, new DailyCleanupJob());
```

Schedulable just fires the work. The actual work usually delegates to Batch or Queueable.

### System.Finalizer

```apex
public class CleanupFinalizer implements System.Finalizer {
    public void execute(FinalizerContext ctx) {
        if (ctx.getResult() == ParentJobResult.UNHANDLED_EXCEPTION) {
            // log failure, send alert, re-enqueue with backoff
            System.debug('Job failed: ' + ctx.getException().getMessage());
        }
    }
}
```

Attach it in your Queueable's execute method:

```apex
public void execute(QueueableContext ctx) {
    System.attachFinalizer(new CleanupFinalizer());
    // rest of your logic
}
```

## Queueable with Callout: the Most Common Pattern

This is what you need when a trigger needs to call an external API.

```apex
public class ExternalSyncJob implements Queueable, Database.AllowsCallouts {

    private List<Id> recordIds;

    public ExternalSyncJob(List<Id> recordIds) {
        this.recordIds = recordIds;
    }

    public void execute(QueueableContext ctx) {
        List<Account> accounts = [
            SELECT Id, Name, ExternalId__c
            FROM Account
            WHERE Id IN :this.recordIds
            WITH SECURITY_ENFORCED
        ];

        for (Account acc : accounts) {
            HttpRequest req = new HttpRequest();
            req.setEndpoint('callout:My_Named_Credential/api/accounts');
            req.setMethod('POST');
            req.setBody(JSON.serialize(acc));

            HttpResponse res = new Http().send(req);
            // handle response
        }
    }
}
```

Enqueue from the trigger handler (after-trigger context only):

```apex
protected override void afterInsert() {
    System.enqueueJob(new ExternalSyncJob(Trigger.newMap.keySet()));
}
```

The trigger can't make the callout directly. Enqueue a Queueable that implements `Database.AllowsCallouts`. That's the pattern.
