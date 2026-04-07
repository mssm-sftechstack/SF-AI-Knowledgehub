# Multi-Tenant Architecture

Understanding Salesforce's multi-tenant model helps you understand why limits exist and how to write scalable code.

---

## The Multi-Tenant Model

Salesforce runs thousands of customer orgs on shared infrastructure. All orgs share:

- Databases
- Application servers
- Storage
- Network bandwidth
- CPU and memory pools

### Single-Tenant vs Multi-Tenant

**Single-Tenant** (traditional): One customer, one database server, one app server, isolated.

```
Company A
└── Dedicated database
└── Dedicated app server
└── Isolated storage
```

Cost: Millions per year. Scaling: Buy bigger hardware.

**Multi-Tenant** (Salesforce): Thousands of customers share the same database and servers.

```
Shared Database Server
├── Company A (org)
├── Company B (org)
├── Company C (org)
└── ... 1000s more
Shared App Server
├── Company A code
├── Company B code
├── Company C code
└── ... runs all orgs' code
```

Cost: Hundreds per month. Scaling: Automatic (Salesforce adds servers).

---

## Why Limits Exist

### Reason 1: Resource Contention (Noisy Neighbor Problem)

If one org writes inefficient code, it can slow down all orgs on the same server.

**Example**: You run an unfiltered SOQL query on 10 million records.

```apex
List<Account> allAccounts = [SELECT Id FROM Account];  // No WHERE clause
```

This query:
- Scans 10 million records on the database
- Consumes database CPU
- Locks tables
- Slows down OTHER orgs' queries on that same database

**Solution**: Governor limits block this at runtime.

```
Limit violated: Apex CPU time exceeded
```

You hit the limit before the query times out other orgs' transactions.

### Reason 2: Fairness

If there's no limit, a large customer with unlimited budget could hog all resources.

```
Org A (slow code): Running 200 SOQL queries
Org B (fast code): Waiting for database response
```

Limits ensure every org gets a fair share.

### Reason 3: Preventing Cascading Failures

If one org locks a table for too long, the entire database can deadlock.

Limits prevent:
- Long-running transactions
- Lock contention
- Cascading timeouts across orgs

---

## Governor Limits Explained

### SOQL Limits

**Query Limit**: 100 queries per transaction

Why? Multiple orgs query simultaneously. If no limit:
- One org runs 1000 queries
- Database becomes saturated
- All orgs' queries slow down
- Timeouts cascade

**Governor prevents this**: You hit 100, get an exception, transaction stops. Database unloaded.

**Row Limit**: 50,000 rows per query

Why? If one query returns 1 million rows:
- Locks read on table
- Network transfer 1 MB of data
- Memory used to hold result set
- Other orgs' queries wait

**Governor prevents this**: Query returns max 50,000, you paginate.

### DML Limits

**DML Limit**: 10,000 records per transaction

Why? Updates on 10,000 records = 10,000 row locks on database.

If no limit:
- One org locks 1 million rows
- Other orgs trying to update those rows wait indefinitely
- Deadlock

**Governor prevents this**: 10,000 limits the blast radius.

### CPU Time Limit

**CPU Limit**: 10 seconds of CPU time per transaction

Why? CPU is shared across servers. If one org uses all CPU:
- Other orgs' code starves for CPU
- Timeouts

**Governor prevents this**: 10-second limit for each transaction.

### Storage Limits

**Data Storage**: 5 GB per 10 user licenses

Why? All orgs' data lives on shared storage. If no limit:
- One org stores 100 GB of files
- Disk fills up
- All orgs can't write data

**Governor prevents this**: Quota limits data per org.

---

## Designing for Scale

### Pattern 1: Bulkified Code

Instead of processing one record:

```apex
// ❌ Slow (processes one at a time)
for (Account acc : accounts) {
  update acc;  // DML inside loop = 1 DML per account
}
```

Process many at once:

```apex
// ✅ Fast (batch update)
update accounts;  // 1 DML for all accounts
```

**Impact**: 200 records: 1 transaction instead of 200. 200x faster.

### Pattern 2: Query Once, Loop Processing

Instead of querying in loop:

```apex
// ❌ Slow (100 accounts = 100 SOQL queries)
for (Account acc : accounts) {
  List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE AccountId = :acc.Id];
}
```

Query once, process in memory:

```apex
// ✅ Fast (1 SOQL query)
List<Opportunity> allOpps = [
  SELECT AccountId, Id FROM Opportunity 
  WHERE AccountId IN :new List<Id>(new Map<Id, Account>(accounts).keySet())
];

Map<Id, List<Opportunity>> oppsByAccount = new Map<Id, List<Opportunity>>();
for (Opportunity opp : allOpps) {
  if (!oppsByAccount.containsKey(opp.AccountId)) {
    oppsByAccount.put(opp.AccountId, new List<Opportunity>());
  }
  oppsByAccount.get(opp.AccountId).add(opp);
}

for (Account acc : accounts) {
  List<Opportunity> opps = oppsByAccount.get(acc.Id) ?? new List<Opportunity>();
  // Process
}
```

**Impact**: 200 accounts: 1 query instead of 200. Database unloaded 200x faster.

### Pattern 3: Batch Jobs for Large Data

Instead of processing 1 million records in one transaction:

```apex
// ❌ Wrong (exceeds DML limit)
List<Account> allAccounts = [SELECT Id FROM Account];
for (Account acc : allAccounts) {
  acc.Status__c = 'Updated';
}
update allAccounts;  // 1 million DML = exceeds limit
```

Use batch jobs:

```apex
// ✅ Right (splits into chunks)
Database.executeBatch(new UpdateAccountsBatch(), 200);

// Batch processes 200 at a time
// 1 million records = 5000 batches
// Each batch is its own transaction with its own governor limits
```

**Impact**: 1 transaction that fails becomes 5000 transactions, most succeed.

---

## Pod Architecture

Salesforce infrastructure is divided into pods. Each pod is a cluster of servers.

### Anatomy of a Pod

```
Pod N (shared infrastructure)
├── Database Server 1
│  ├── Org A data
│  ├── Org B data
│  └── Org C data
├── Database Server 2 (replica)
├── App Server 1
│  ├── Org A code
│  ├── Org B code
│  └── Org C code
├── App Server 2
├── Storage Server
├── Cache Server
└── Search Server
```

When you execute Apex in your org:

1. Request routes to an App Server on your pod
2. App Server runs your code
3. Code queries Database Server on same pod
4. Results returned to you

### Pod Affinity

Your org stays on the same pod (mostly). Some migrating happens, but:

- Your data is on Pod 5
- Your app server requests go to Pod 5
- Your queries hit Pod 5's database

This keeps latency low.

---

## Resource Sharing at Scale

### Example: 1000 Orgs on Pod 5

```
Pod 5 Database
├── Org A: 10 GB data, 100 QPS (queries per second)
├── Org B: 5 GB data, 50 QPS
├── Org C: 2 GB data, 10 QPS
└── ... 997 more orgs
Total: ~100 TB shared database
```

All queries from all orgs hit the same database. If Org A runs an unfiltered query:

```apex
[SELECT * FROM Account]  // No WHERE clause
```

This scans:
- Org A's 10M accounts
- Locks the database for Org A
- Blocks Org B's query
- Blocks Org C's query

Orgs B and C time out waiting for database.

**Governor limit prevents this**: Org A hits the limit before the lock becomes a problem.

---

## Planning for Growth

### What Changes as You Scale

| Metric | Dev Org | Prod Org | Enterprise |
|--------|---------|----------|-----------|
| Records | < 1M | 1-100M | 100M+ |
| Daily Transactions | < 1000 | 10K-100K | 100K+ |
| Users | < 50 | 50-500 | 500+ |
| Custom Fields | < 200 | 200-500 | 500+ |

### Query Performance Degrades at Scale

A query that runs in 100ms with 1M records may run in 5s with 100M records.

**Solution**: Use selective filters (WHERE clause with indexed fields).

```apex
// ❌ Slow at scale (scans all records)
List<Account> accounts = [SELECT Id FROM Account];

// ✅ Fast at scale (uses index on Status__c)
List<Account> activeAccounts = [SELECT Id FROM Account WHERE Status__c = 'Active'];
```

### DML Slows Down

Inserting 10,000 records takes time. With 100M existing records, insert locks more, slower.

**Solution**: Batch into smaller chunks.

```apex
// ✅ Fast (smaller batches)
Database.executeBatch(new InsertBatch(), 200);  // 200 per transaction
```

---

## Multi-Tenant Awareness in Code

### Anti-Pattern 1: Assuming Your Data is Small

```apex
// ❌ Assumes < 1000 accounts
List<Account> accounts = [SELECT Id FROM Account];  // Could be 100M records!
```

### Anti-Pattern 2: Unbounded Loops

```apex
// ❌ Assumes loop will finish in time
for (Integer i = 0; i < 1000000; i++) {
  // Process
}
// Hits CPU limit at 10 seconds
```

### Anti-Pattern 3: Assuming Sync is Fast Enough

```apex
// ❌ Assumes sync call completes fast
callExternalAPI();  // If API slow, user waits > 30 seconds, times out
update accounts;
```

### Correct Pattern 1: Always Selective

```apex
// ✅ Selective (uses WHERE, fast)
List<Account> activeAccounts = [
  SELECT Id FROM Account 
  WHERE Status__c = 'Active'
];
```

### Correct Pattern 2: Plan for Iteration

```apex
// ✅ Handles 100M records
Integer pageSize = 10000;
Decimal nextOffset = 0;

do {
  List<Account> accounts = [
    SELECT Id FROM Account 
    WHERE Status__c = 'Active'
    LIMIT :pageSize
    OFFSET :nextOffset
  ];
  
  // Process accounts
  nextOffset += pageSize;
} while (accounts.size() == pageSize);  // Stop when query returns < pageSize
```

### Correct Pattern 3: Defer to Async

```apex
// ✅ Async (user doesn't wait)
System.enqueueJob(new ProcessAccountsQueueable());
// User gets "Processing in background" message
```

---

## Key Takeaways

1. **Limits are Features**: They protect all orgs from noisy neighbors. Design within them, don't fight them.

2. **Scale Happens Gradually**: Your org starts small (1M records). Code that works for 1M may break at 100M. Plan ahead.

3. **Bulkify Everything**: Process 200+ records at once. One bulk DML is faster and cheaper than 200 individual ones.

4. **Indexes Are Your Friend**: Use WHERE clauses with indexed fields (Name, Id, standard fields, fields with external IDs).

5. **Test at Scale**: Bulk test with 200+ records. Catch limits in dev, not production.

6. **Defer to Async**: Long operations = Batch Apex, not synchronous. Let Salesforce schedule them.

7. **Database is Shared**: Your org shares database with 1000s of others. Respect that. Query selectively. Update efficiently.

---

## Checklist: Multi-Tenant Aware Code

- ✅ All queries have WHERE clause (selective)
- ✅ No SOQL/DML in loops
- ✅ Batch for 200+ records
- ✅ Bulk tested (200+ records)
- ✅ CPU time monitored (< 10s)
- ✅ Storage aware (data culled regularly)
- ✅ Async used for long operations (Batch, Queueable)
- ✅ Code assumes data can be large (100M+ records)
- ✅ No unbounded loops or iterations
- ✅ External callouts deferred (not blocking)

