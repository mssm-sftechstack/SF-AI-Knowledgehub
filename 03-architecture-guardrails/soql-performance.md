# SOQL Performance

**A slow SOQL query in dev becomes a 30-second page load or a CPU limit exception in production. Know what makes a query fast before you write it.**

## What Makes a Query Selective

Selectivity is about what percentage of rows your WHERE clause matches. Salesforce uses a threshold to decide whether to use an index or do a full table scan.

For standard objects: if your filter matches more than 10% of rows, Salesforce may skip the index and scan the whole table. For large objects (1M+ rows), even 1% can be 10,000 rows and slow.

A query is selective when it filters on indexed fields and the filter is narrow enough that the index scan is cheaper than a table scan.

## Standard Indexed Fields

These fields are indexed automatically on every object:

| Field | Notes |
|---|---|
| `Id` | Primary key, always indexed |
| `Name` | Indexed on most objects |
| `OwnerId` | Indexed |
| `CreatedDate` | Indexed |
| `LastModifiedDate` | Indexed |
| `RecordTypeId` | Indexed |
| Master-detail / lookup fields | Indexed (the relationship field) |
| External ID fields | Indexed when marked as External ID |
| Unique fields | Indexed automatically |

Custom fields aren't indexed unless you mark them as External ID, Unique, or request a custom index via Salesforce Support.

## Anti-Patterns

| Pattern | Problem | Fix |
|---|---|---|
| `WHERE Name LIKE '%search'` (leading wildcard) | Full table scan, index can't help | Use `LIKE 'search%'` or a custom index on a separate field |
| `WHERE CustomField__c = 'value'` with no indexed field | Full table scan on large objects | Add an indexed field to the filter (`AND OwnerId = :userId`) |
| No LIMIT clause | Returns every matching row, blows heap or row limits | Always add `LIMIT` appropriate to the use case |
| `SELECT Id, Field1, Field2, ..., all 50 fields` | Huge heap usage | Select only the fields you use |
| SOQL inside a loop | Hits 101 query limit fast | Collect IDs first, query once outside the loop |

### **Anti-Pattern**: SOQL in a loop

```apex
// WRONG
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
    // processes contacts
}

// RIGHT
Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();
List<Contact> contacts = [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds];
Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
for (Contact c : contacts) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}
```

## Relationship Queries

### Parent dot-notation (child to parent)

```apex
// Traverse up to parent fields — one hop
SELECT Id, Name, Account.Name, Account.OwnerId
FROM Contact
WHERE Account.Industry = 'Technology'
WITH SECURITY_ENFORCED
```

No extra query cost for parent traversal. Resolved in the same query.

### Child subquery (parent to children)

```apex
// Fetch children inline — counts as one of your 20 subquery slots
SELECT Id, Name,
    (SELECT Id, LastName, Email FROM Contacts LIMIT 200),
    (SELECT Id, Subject FROM Cases WHERE Status = 'Open')
FROM Account
WHERE Id IN :accountIds
WITH SECURITY_ENFORCED
```

Limits to know:
- Max 20 subqueries per query
- Child subqueries can only go one level deep (no subquery inside a subquery)
- Each subquery returns up to 50k rows but counts against your total row limit

## WITH SECURITY_ENFORCED vs WITH USER_MODE

| Keyword | Where it applies | What it does on violation |
|---|---|---|
| `WITH SECURITY_ENFORCED` | SELECT and WHERE fields | Throws `System.QueryException` if any field is inaccessible |
| `WITH USER_MODE` (API 56.0+) | Full DML + SOQL, FLS + CRUD | Strips inaccessible fields silently on read, throws on CRUD violation |

Use `WITH SECURITY_ENFORCED` on read-only Selector queries where you want explicit failure if a field is missing access. Use `WITH USER_MODE` for write operations in service classes where you want the platform to enforce full FLS and CRUD at the database level.

```apex
// Read query — fail loudly if field access is missing
public static List<Account> getAccountsForReport(Set<Id> ids) {
    return [
        SELECT Id, Name, AnnualRevenue, BillingCountry
        FROM Account
        WHERE Id IN :ids
        WITH SECURITY_ENFORCED
    ];
}

// Write operation — platform enforces FLS + CRUD
public static void updateAccountsAsUser(List<Account> accounts) {
    update as user accounts; // WITH USER_MODE equivalent for DML
}
```

When using `Security.stripInaccessible()` instead of `WITH SECURITY_ENFORCED`, the query runs without the keyword and you strip fields after:

```apex
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Name, AnnualRevenue FROM Account WHERE Id IN :ids]
);
List<Account> safeAccounts = (List<Account>) decision.getRecords();
```

Use `stripInaccessible()` when you want to return partial data rather than throw an exception.
