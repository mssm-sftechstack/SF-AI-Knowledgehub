---
name: sf-soql
description: >
  SOQL authoring, query optimisation, index usage, selective filters, relationship queries,
  aggregate functions, query plan analysis, and anti-patterns. Based on Salesforce SOQL
  and SOSL Reference (API 62.0).
allowed-tools: Bash(sf apex run*), Bash(sf data query*)
---

# Salesforce SOQL Skill

**SOQL authoring, query optimisation, index usage, selective filters, relationship queries, aggregate functions, and anti-patterns.**

## 1. When SOQL Runs Efficiently

A query is efficient when it hits an index and is selective. Salesforce skips the full table scan when:

- The WHERE clause filters on an indexed field
- The filter is **selective** (returns a small enough fraction of records)
- No leading wildcard is used in LIKE patterns

Selectivity threshold (approximate):
- Objects under 100,000 records: filter must return fewer than 10% of total records
- Objects with 100,000+ records: filter must return fewer than 333,333 records OR less than 10%, whichever is smaller
- SystemModstamp and CreatedDate filters with recent date ranges are almost always selective on large objects

```apex
// efficient - filters on indexed field, bound variable, selective result
List<Opportunity> opps = [
    SELECT Id, Name, Amount, StageName
    FROM Opportunity
    WHERE OwnerId = :userId
      AND StageName = 'Prospecting'
      AND CloseDate >= :startDate
    WITH SECURITY_ENFORCED
    LIMIT 200
];

// inefficient - LIKE with leading wildcard forces full table scan
List<Account> accs = [
    SELECT Id, Name
    FROM Account
    WHERE Name LIKE '%Corp'
    WITH SECURITY_ENFORCED
];
```

> **Order of Execution and Limit Counting**
> SOQL limits count across the entire transaction, not just your code. Before-triggers, after-triggers, Flows, workflow field updates, and any code they invoke all share the same 100-query limit. A before-trigger that runs 10 queries plus an after-save Flow that runs 5 leaves you with 85. Design your SOQL budget assuming other automations consume some of it.

---

## 2. Index Types

| Field | Indexed | Notes |
|---|---|---|
| Id | Yes | Primary key |
| Name | Yes | Standard indexed |
| OwnerId | Yes | Lookup to User |
| CreatedDate | Yes | Use for time-range filters |
| LastModifiedDate | Yes | Use for delta syncs |
| SystemModstamp | Yes | Preferred for replication |
| RecordTypeId | Yes | Standard indexed |
| Master-detail fields | Yes | FK always indexed |
| Lookup fields | Yes | FK always indexed |
| External IDs | Yes | Custom fields marked External ID |
| Unique custom fields | Yes | Custom fields marked Unique |

Custom fields NOT marked External ID or Unique are NOT indexed. To request a custom index, open a Salesforce Support case.

---

## 3. Query Anti-Patterns

| Anti-Pattern | Example | Problem | Fix |
|---|---|---|---|
| Leading wildcard LIKE | `WHERE Name LIKE '%Corp'` | Full table scan | Trailing wildcard or switch to SOSL |
| Non-indexed filter only | `WHERE Description__c = :val` | Full table scan | Add indexed field to filter |
| SOQL in a for loop | Query inside `for(SObject s : list)` | Hits 101 SOQL limit | Move query outside loop, use map |
| No LIMIT on large objects | `SELECT Id FROM Log__c` | Returns 50k rows | Always add LIMIT |
| SELECT all fields | `FIELDS(ALL)` | Expensive, brittle | Select only required fields |
| Cartesian subquery | Subquery returning >1000 rows | Inner query limit 1000 | Filter subquery, split if needed |

---

## 4. Relationship Queries

### Parent-to-Child (subquery)

```apex
List<Account> accs = [
    SELECT Id, Name,
        (SELECT Id, Name, Amount FROM Opportunities
         WHERE StageName != 'Closed Lost'
         ORDER BY CloseDate ASC
         LIMIT 50)
    FROM Account
    WHERE Id IN :accountIds
    WITH SECURITY_ENFORCED
];

for (Account a : accs) {
    for (Opportunity o : a.Opportunities) {
        System.debug(o.Name + ': ' + o.Amount);
    }
}
```

### Child-to-Parent (dot notation)

```apex
List<Contact> contacts = [
    SELECT Id, FirstName, LastName,
           Account.Name, Account.BillingCity
    FROM Contact
    WHERE Account.Type = 'Customer'
      AND AccountId != null
    WITH SECURITY_ENFORCED
    LIMIT 200
];
```

### Relationship Query Limits

| Limit | Value |
|---|---|
| Subqueries per query | 20 |
| Depth of subquery nesting | 1 level |
| Rows per subquery per parent record | 1,000 |
| Relationship traversal (dot notation) | 5 levels |

---

## 5. Aggregate Functions

```apex
// GROUP BY with HAVING
AggregateResult[] highVolume = [
    SELECT AccountId, COUNT(Id) oppCount, SUM(Amount) totalAmt
    FROM Opportunity
    WHERE IsWon = true
    WITH SECURITY_ENFORCED
    GROUP BY AccountId
    HAVING COUNT(Id) > 5
];

for (AggregateResult ar : highVolume) {
    Id acctId = (Id) ar.get('AccountId');
    Integer cnt = (Integer) ar.get('oppCount');
}
```

---

## 6. Dynamic SOQL

```apex
// mandatory: escape user-supplied strings
String safeInput = String.escapeSingleQuotes(rawInput);
String soql = 'SELECT Id, Name FROM Account WHERE Name = \'' + safeInput + '\'';
List<Account> results = Database.query(soql);

// bound variable in dynamic SOQL -- variable must be in scope
String fieldFilter = 'Prospecting';
String soql2 = 'SELECT Id, Name FROM Opportunity WHERE StageName = :fieldFilter';
List<Opportunity> opps = Database.query(soql2);
```

Dynamic SOQL does NOT support `WITH SECURITY_ENFORCED` as a string literal. Use `Security.stripInaccessible()` post-query or switch to USER_MODE.

---

## 7. WITH USER_MODE

Introduced in API 56.0. Enforces FLS and CRUD at the database level. Preferred for new service class development.

```apex
List<Account> accs = [
    SELECT Id, Name, Revenue__c
    FROM Account
    WHERE OwnerId = :UserInfo.getUserId()
    WITH USER_MODE
    LIMIT 200
];

Database.insert(newRecords, AccessLevel.USER_MODE);
Database.update(updatedRecords, AccessLevel.USER_MODE);
```

| Method | FLS Read | CRUD Read | FLS Write | CRUD Write | Works in Dynamic SOQL |
|---|---|---|---|---|---|
| WITH SECURITY_ENFORCED | Yes | Yes | No | No | No |
| Security.stripInaccessible() | Yes | No | Yes | No | Yes |
| WITH USER_MODE | Yes | Yes | Yes | Yes | Yes |

---

## 8. SOQL Limits

| Limit | Value |
|---|---|
| SOQL queries per synchronous transaction | 100 |
| SOQL queries per asynchronous transaction | 200 |
| Total rows returned per transaction | 50,000 |
| Subqueries per parent query | 20 |
| Rows per subquery per parent row | 1,000 |
| Query timeout | 120 seconds |

---

## 9. Checklist

- [ ] Filters on at least one indexed field
- [ ] Result set is selective (fewer than ~10% of total records on large objects)
- [ ] No leading wildcard in any LIKE clause
- [ ] LIMIT present
- [ ] `WITH SECURITY_ENFORCED` or `WITH USER_MODE` present
- [ ] Query is outside all loops
- [ ] Bound variables used -- no string concatenation of user input
- [ ] `SELECT *` / `FIELDS(ALL)` is NOT used
- [ ] Result assigned to `List<>` -- never directly to a single SObject without isEmpty() guard
- [ ] Test asserts exact row count, not just "no exception thrown"

---

## Sources

- [SOQL and SOSL Reference](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/)
- [Query and Search Optimization Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.salesforce_query_optimization.meta/salesforce_query_optimization/)
- [USER_MODE and AccessLevel](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_enforce_usermode.htm)
