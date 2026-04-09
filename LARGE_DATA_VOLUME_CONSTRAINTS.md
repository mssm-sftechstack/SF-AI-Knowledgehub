# Large Data Volume (LDV) Architecture

**CRITICAL DIRECTIVE FOR AI:** If an object contains more than 1 million records, standard Apex and SOQL practices will fail due to database lock timeouts and full table scans. You must write code optimized for Large Data Volumes.

## 1. Selective SOQL (Index Forcing)
If querying an LDV object, the SOQL query MUST be selective.
* **The Rule:** The `WHERE` clause MUST contain at least one indexed field (e.g., `Id`, `Name`, `CreatedDate`, or a custom field marked as an External ID / Unique).
* NEVER use leading wildcards in a `LIKE` clause (e.g., `LIKE '%Smith'`), as this bypasses the database index and forces a full table scan.

```apex
// ❌ DANGEROUS: Full table scan on millions of records
List<Task> tasks = [SELECT Id FROM Task WHERE Status = 'Open'];

// ✅ SECURE: Uses an indexed field to filter the dataset
List<Task> tasks = [SELECT Id FROM Task WHERE AccountId IN :accountIds AND Status = 'Open'];
```

## 2. The SOQL For-Loop Heap Bypass
When processing up to 50,000 records in a batch or asynchronous job, assigning the query results to a single List will blow up the 6MB Heap Size limit.
* **The Rule:** You must use a SOQL For-Loop to chunk the records in memory.

```apex
// ❌ DANGEROUS: Loads 50,000 records into memory at once
List<Account> allAccounts = [SELECT Id, Name FROM Account WHERE CreatedDate = TODAY];
for(Account acc : allAccounts) { ... }

// ✅ SECURE: Processes records in heap-safe chunks of 200
for(List<Account> accountChunk : [SELECT Id, Name FROM Account WHERE CreatedDate = TODAY]) {
    for(Account acc : accountChunk) {
        // Process data
    }
}
```

## 3. Account Data Skew Prevention
If a single parent record (like an Account) has more than 10,000 child records, updating those children concurrently will cause the parent record to lock (`UNABLE_TO_LOCK_ROW`).
* **The Rule:** When writing Batch Apex to update child records, you must `ORDER BY ParentId` in your `start()` method query to ensure all children of the same parent are processed in the same execution chunk, preventing cross-chunk locking contention.