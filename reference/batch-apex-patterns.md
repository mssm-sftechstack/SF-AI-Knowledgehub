# Batch Apex Patterns

Decision tree for async processing and batch job examples.

---

## Decision Tree: Which Async Pattern?

```
How many records to process?

├─ < 10,000 and need result immediately
│  └─ Use: Queueable
│
├─ 10,000 - 1,000,000
│  └─ Use: Batch Apex
│
├─ Need to run every day/week/month
│  └─ Use: Scheduled Apex + Batch
│
└─ Need real-time notifications
   └─ Use: Platform Events
```

### Queueable (< 10,000 records)

**When**: You need to process up to 10,000 records and don't mind async delay.

**Pros**:
- Easy to write (similar to normal methods)
- Can chain into itself
- Allows callouts
- Result available in next transaction

**Cons**:
- Max 10,000 records per job
- Waits in queue before execution

```apex
System.enqueueJob(new ProcessAccountsQueueable(new List<Account>{...}));
```

### Batch Apex (10,000+ records)

**When**: Processing 10,000+ records.

**Pros**:
- Handles millions of records
- Each batch is isolated transaction
- Governor limits reset per batch
- Can schedule to run at specific time

**Cons**:
- Slower (processes in batches)
- Result not immediately available
- No callouts (but can enqueue Queueable from batch)

```apex
Database.executeBatch(new ProcessAccountsBatch(), 200);  // 200 records per batch
```

### Scheduled Apex (recurring jobs)

**When**: Job needs to run daily, weekly, monthly.

**Pros**:
- Runs at specific time
- Can enqueue Batch jobs

**Cons**:
- No callouts from schedulable context (use Queueable)
- Fixed schedule, not dynamic

```apex
System.schedule('Job Name', '0 0 2 * * ?', new MySchedulableClass());  // 2 AM daily
```

### Platform Events (real-time async)

**When**: Need immediate async execution (not queued).

**Pros**:
- Published immediately
- Subscribers get event in next transaction
- Loosely coupled (publisher doesn't know subscriber)

**Cons**:
- Not guaranteed delivery (if subscriber fails, event lost)
- Limited message size

```apex
EventBus.publish(new Account_Updated__e(accountId = acc.Id));
```

---

## Queueable Pattern

```apex
public class ProcessAccountsQueueable implements Queueable {
  private List<Account> accounts;
  
  public ProcessAccountsQueueable(List<Account> accounts) {
    this.accounts = accounts;
  }
  
  public void execute(QueueableContext context) {
    for (Account acc : accounts) {
      acc.Status__c = 'Processed';
    }
    update accounts;
  }
}

// Usage
System.enqueueJob(new ProcessAccountsQueueable(accounts));
```

### Queueable with Callouts

```apex
public class CallExternalAPIQueueable implements Queueable, Database.AllowsCallouts {
  public void execute(QueueableContext context) {
    HttpRequest req = new HttpRequest();
    req.setEndpoint('https://api.example.com/users');
    req.setMethod('GET');
    
    Http http = new Http();
    HttpResponse res = http.send(req);
    
    if (res.getStatusCode() == 200) {
      System.debug('Success: ' + res.getBody());
    }
  }
}

// Usage
System.enqueueJob(new CallExternalAPIQueueable());
```

### Queueable Chaining

```apex
public class ChainableQueueable implements Queueable {
  private Integer batchNumber;
  private static final Integer MAX_BATCHES = 5;
  
  public ChainableQueueable(Integer batchNumber) {
    this.batchNumber = batchNumber;
  }
  
  public void execute(QueueableContext context) {
    System.debug('Processing batch ' + batchNumber);
    
    // Do work
    
    // Chain next job
    if (batchNumber < MAX_BATCHES) {
      System.enqueueJob(new ChainableQueueable(batchNumber + 1));
    }
  }
}

// Usage
System.enqueueJob(new ChainableQueueable(1));  // Starts chain
```

---

## Batch Apex Pattern

```apex
public class UpdateAccountsBatch implements Database.Batchable<SObject> {
  public Database.QueryLocator start(Database.BatchableContext context) {
    // Query returns data for batch to process
    return Database.getQueryLocator('SELECT Id, Name FROM Account WHERE Status__c = \'Active\'');
  }
  
  public void execute(Database.BatchableContext context, List<Account> scope) {
    // scope contains up to 200 records (batch size)
    for (Account acc : scope) {
      acc.Name = acc.Name + ' (Updated)';
    }
    update scope;
  }
  
  public void finish(Database.BatchableContext context) {
    // Runs after all batches complete
    System.debug('Batch job finished');
    
    // Can enqueue another job here
    System.enqueueJob(new NotificationQueueable());
  }
}

// Usage
Database.executeBatch(new UpdateAccountsBatch(), 200);  // 200 records per batch
```

### Stateful Batch (Retains State Across Batches)

```apex
public class StatefulBatch implements Database.Batchable<SObject>, Database.Stateful {
  public Integer recordsProcessed = 0;
  
  public Database.QueryLocator start(Database.BatchableContext context) {
    return Database.getQueryLocator('SELECT Id FROM Account');
  }
  
  public void execute(Database.BatchableContext context, List<Account> scope) {
    recordsProcessed += scope.size();
    update scope;
  }
  
  public void finish(Database.BatchableContext context) {
    System.debug('Total records processed: ' + recordsProcessed);
  }
}
```

Without `Database.Stateful`, each batch gets a fresh instance. With it, recordsProcessed persists.

### Batch with Error Handling

```apex
public class RobustBatch implements Database.Batchable<SObject> {
  public Database.QueryLocator start(Database.BatchableContext context) {
    return Database.getQueryLocator('SELECT Id FROM Account');
  }
  
  public void execute(Database.BatchableContext context, List<Account> scope) {
    List<Account> toUpdate = new List<Account>();
    
    for (Account acc : scope) {
      try {
        acc.Status__c = 'Processed';
        toUpdate.add(acc);
      } catch (Exception ex) {
        System.debug('Error processing account ' + acc.Id + ': ' + ex.getMessage());
        // Continue processing others
      }
    }
    
    if (!toUpdate.isEmpty()) {
      try {
        update toUpdate;
      } catch (DmlException ex) {
        System.debug('DML Error: ' + ex.getMessage());
      }
    }
  }
  
  public void finish(Database.BatchableContext context) {
    // Send notification
  }
}
```

---

## Scheduled Apex Pattern

```apex
global class RefreshCoverageSchedulable implements Schedulable {
  global void execute(SchedulableContext context) {
    System.debug('Scheduled job running');
    
    // Enqueue batch job
    Database.executeBatch(new RefreshCoverageBatch(), 200);
  }
}

// Schedule the job
System.schedule('Refresh Coverage Daily', '0 0 2 * * ?', new RefreshCoverageSchedulable());
```

### Cron Expression Format

```
Minute Hour DayOfMonth Month DayOfWeek
0      0    *          *     ?           = Midnight every day
0      2    *          *     ?           = 2 AM every day
0      9    *          *     1-5         = 9 AM weekdays
0      0    1          *     ?           = Midnight on 1st of month
*/15   *    *          *     ?           = Every 15 minutes
```

### Testing Scheduled Apex

```apex
@IsTest
static void testScheduledApex() {
  Test.startTest();
  
  // Don't use System.schedule() if your schedulable enqueues Queueable with callout
  // Instead, call execute() directly
  RefreshCoverageSchedulable schedulable = new RefreshCoverageSchedulable();
  schedulable.execute(null);  // Pass null for SchedulableContext in test
  
  Test.stopTest();
  
  // Verify results
}
```

---

## Anti-Patterns

### ❌ Don't: SOQL in Loop in Batch

```apex
// WRONG
public void execute(Database.BatchableContext context, List<Account> scope) {
  for (Account acc : scope) {
    List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE AccountId = :acc.Id];
    // Process opps
  }
}

// RIGHT
public Database.QueryLocator start(Database.BatchableContext context) {
  return Database.getQueryLocator(
    'SELECT Id, (SELECT Id FROM Opportunities) FROM Account'
  );
}

public void execute(Database.BatchableContext context, List<Account> scope) {
  for (Account acc : scope) {
    List<Opportunity> opps = acc.Opportunities;
    // Process opps
  }
}
```

### ❌ Don't: No Batch Size Limit

```apex
// WRONG
Database.executeBatch(new MyBatch());  // Default 200, might hit limits

// RIGHT
Database.executeBatch(new MyBatch(), 100);  // Explicit smaller batch for safety
```

### ❌ Don't: Assume finish() Runs Immediately

```apex
// WRONG
Database.executeBatch(new MyBatch());
// Immediately query results
List<Account> updated = [SELECT Id FROM Account WHERE Updated__c = true];

// RIGHT
// Batch is async, runs later. Don't assume synchronous completion.
```

---

## Checklist: Batch Apex Ready for Production

- ✅ start() returns QueryLocator (data bounded)
- ✅ execute() handles batch of records (not individual)
- ✅ Error handling in execute() (try-catch)
- ✅ finish() does cleanup or chains next job
- ✅ Batch size set explicitly (not default)
- ✅ Bulk tested (multiple batches)
- ✅ No SOQL in loops in execute()
- ✅ Stateful only if state needed
- ✅ Schedulable has execute() method
- ✅ Cron expression tested

