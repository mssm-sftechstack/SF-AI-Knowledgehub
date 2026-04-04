# Governor Limits Quick Reference

**The hard caps Salesforce enforces per transaction. Design for the target, not the ceiling.**

---

## Synchronous Limits

| Limit | Per Transaction | Design Target |
|---|---|---|
| SOQL queries | 100 | Under 20. If you're near 50, something is wrong. |
| SOQL rows returned | 50,000 | Query with selective filters. Never SELECT * in bulk operations. |
| DML statements | 150 | Under 10. Bulkify: one insert for a list, not one per record. |
| DML rows | 10,000 | Batch your updates. With 200-record trigger batches, this is fine. |
| CPU time | 10,000 ms | Under 5,000 ms in normal code. Avoid loops inside loops. |
| Heap size | 6 MB | Watch for large JSON blobs and large list-of-lists. |
| Callouts per transaction | 100 | Async context only. Never more than 10 per Queueable job in practice. |
| Callout timeout | 120 seconds (max) | Set explicitly. Match the API's SLA. |
| Future methods per transaction | 50 | Prefer Queueable for chaining and better error handling. |
| String length | 6 MB | Heap-counted. Large string operations count against heap. |

---

## Async-Specific Limits

| Context | Limit | Notes |
|---|---|---|
| Batch Apex scope | 1 to 2,000 per batch | Default 200. Max 2,000. Set based on query row count and CPU cost per record. |
| Batch Apex concurrent jobs | 5 active jobs | More jobs queue. Active = running execute() right now. |
| Queueable job depth | 50 chained jobs | A Queueable can enqueue one more job. That job can enqueue one more. Max chain depth: 50. |
| @future per transaction | 50 | Counted against the transaction that calls them, not the future context. |
| Platform Events per hour | 250,000 (publish) | Subscriber limit varies by license. Check Salesforce Event Monitoring. |
| Scheduled Apex jobs | 100 scheduled | Across the entire org. Active Batch Apex jobs count against this. |
| Email invocations | 10 per transaction | Applies to `Messaging.sendEmail()`. Not email alerts in Flow. |

---

## Key Design Principles

**Never put SOQL inside a loop.**
```apex
// WRONG: 200 SOQL queries for 200 records
for (Account acc : accounts) {
    List<Case> cases = [SELECT Id FROM Case WHERE AccountId = :acc.Id]; // one query per iteration
}

// RIGHT: one query, map the results
Map<Id, List<Case>> casesByAccount = new Map<Id, List<Case>>();
for (Case c : [SELECT Id, AccountId FROM Case WHERE AccountId IN :accountIds]) {
    if (!casesByAccount.containsKey(c.AccountId)) casesByAccount.put(c.AccountId, new List<Case>());
    casesByAccount.get(c.AccountId).add(c);
}
```

**Never put DML inside a loop.**
```apex
// WRONG: 200 DML statements
for (Account acc : accounts) {
    acc.Rating = 'Hot';
    update acc; // one DML per iteration
}

// RIGHT: one DML
for (Account acc : accounts) {
    acc.Rating = 'Hot';
}
update accounts;
```

**CPU time is invisible until it isn't.** String concatenation in a loop is a common CPU killer. Use `List<String>` + `String.join()` instead of `str += value` in a loop.
