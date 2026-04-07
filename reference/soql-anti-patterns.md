# SOQL Anti-Patterns & Optimization

Common SOQL mistakes and how to fix them.

---

## Anti-Pattern 1: SELECT *

### Wrong

```apex
List<Account> accounts = [SELECT * FROM Account];
```

**Problem**: Selects all fields. Some may be:
- Inaccessible (FLS violation if stripped)
- Large (text areas slow query)
- Unneeded (wastes memory)

### Right

```apex
List<Account> accounts = [SELECT Id, Name, Revenue__c FROM Account];
```

Select only needed fields.

---

## Anti-Pattern 2: No WHERE Clause

### Wrong

```apex
List<Account> allAccounts = [SELECT Id FROM Account];
```

With 100M records, this scans the entire table.

### Right

```apex
List<Account> activeAccounts = [SELECT Id FROM Account WHERE Status__c = 'Active'];
```

Use WHERE with an indexed field (Name, Id, Status, etc.).

---

## Anti-Pattern 3: SOQL in Loops

### Wrong

```apex
for (Account acc : accounts) {
  List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE AccountId = :acc.Id];
  // Process opps
}
```

200 accounts = 200 SOQL queries. Limit is 100.

### Right

```apex
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
  // Process opps
}
```

1 SOQL query, then process in memory.

---

## Anti-Pattern 4: Aggregate in Code

### Wrong

```apex
List<Opportunity> opps = [SELECT Amount FROM Opportunity WHERE AccountId = :accId];
Decimal total = 0;
for (Opportunity opp : opps) {
  total += opp.Amount;
}
```

Fetches all records to memory, then sums.

### Right

```apex
AggregateResult ar = [
  SELECT SUM(Amount) totalAmount FROM Opportunity WHERE AccountId = :accId
];
Decimal total = (Decimal) ar.get('totalAmount');
```

Database sums, returns 1 result.

---

## Anti-Pattern 5: No Index Awareness

### Wrong

```apex
List<Account> accounts = [SELECT Id FROM Account WHERE CustomField__c = 'Value'];
```

If CustomField__c has no index, this scans all records.

### Right

```apex
List<Account> accounts = [
  SELECT Id FROM Account 
  WHERE Status__c = 'Active'  // Status has index
];
```

Use indexed fields in WHERE (Name, Email, Status, custom fields with External ID).

---

## Anti-Pattern 6: Complex JOINs with Subqueries

### Wrong (Complex to Read)

```apex
List<Account> accounts = [
  SELECT Id, (SELECT Id FROM Opportunities), (SELECT Id FROM Contacts)
  FROM Account
];
```

Subqueries are expensive. Each Account gets multiple subquery results.

### Right (Separate Queries)

```apex
List<Account> accounts = [SELECT Id FROM Account];

List<Opportunity> opps = [
  SELECT AccountId, Id FROM Opportunity 
  WHERE AccountId IN :new List<Id>(new Map<Id, Account>(accounts).keySet())
];

List<Contact> contacts = [
  SELECT AccountId, Id FROM Contact
  WHERE AccountId IN :new List<Id>(new Map<Id, Account>(accounts).keySet())
];
```

Separate queries, easier to understand and debug.

---

## Anti-Pattern 7: Inequality Operators

### Wrong (slow with large data)

```apex
List<Account> accounts = [SELECT Id FROM Account WHERE Status__c != 'Inactive'];
```

`!=` is slow. Database can't use index efficiently.

### Right (use positive logic)

```apex
List<Account> activeAccounts = [
  SELECT Id FROM Account 
  WHERE Status__c = 'Active' OR Status__c = 'Pending'
];
```

Positive WHERE clauses use indexes.

---

## Anti-Pattern 8: LIKE Without Wildcards

### Wrong

```apex
List<Account> results = [SELECT Id FROM Account WHERE Name LIKE 'Acme'];
```

Matches "Acme" exactly. Same as `=`. LIKE is slower.

### Right

```apex
List<Account> results = [SELECT Id FROM Account WHERE Name LIKE 'Acme%'];
```

`%` means wildcard. Matches "Acme Corp", "Acme Inc", etc.

---

## Anti-Pattern 9: NOT IN with Large List

### Wrong

```apex
List<Id> excludeIds = /* 1000 IDs */;
List<Account> results = [SELECT Id FROM Account WHERE Id NOT IN :excludeIds];
```

`NOT IN` with large lists is slow.

### Right

```apex
List<Id> includeIds = /* 100 IDs you want */;
List<Account> results = [SELECT Id FROM Account WHERE Id IN :includeIds];
```

Use `IN` instead of `NOT IN`.

---

## Anti-Pattern 10: Dynamic SOQL with Concatenation

### Wrong (XSS/Injection Risk)

```apex
String searchName = userInput;  // 'Test\' OR Name != NULL; --'
String query = 'SELECT Id FROM Account WHERE Name = \'' + searchName + '\'';
List<Account> results = Database.query(query);  // SOQL injection!
```

### Right (Bind Variables)

```apex
String searchName = userInput;
List<Account> results = [SELECT Id FROM Account WHERE Name = :searchName];
```

Bind variables (`:variable`) are injection-safe.

---

## Performance Tips

### Index Usage

Indexes exist on:
- Id (always)
- Name (always)
- Email (always on Email field)
- Standard fields (Status, Type, etc.)
- Fields marked External ID
- Custom fields with External ID

**Query uses index if**:
- WHERE clause filters on indexed field
- Filter is equality (=) not inequality (!=)
- Filter is NOT negated (WHERE Status = 'Active' ✓, WHERE Status != 'Active' ✗)

### Selective Queries

```apex
// Selective (uses index on Status__c)
[SELECT Id FROM Account WHERE Status__c = 'Active' AND Name = 'Acme']

// Not selective (scans all because of LIKE)
[SELECT Id FROM Account WHERE Status__c = 'Active' AND Name LIKE 'Acme%']
```

### Limit and Offset for Pagination

```apex
// Page 1 (records 0-99)
List<Account> page1 = [SELECT Id FROM Account LIMIT 100 OFFSET 0];

// Page 2 (records 100-199)
List<Account> page2 = [SELECT Id FROM Account LIMIT 100 OFFSET 100];

// Page 3 (records 200-299)
List<Account> page3 = [SELECT Id FROM Account LIMIT 100 OFFSET 200];
```

Avoid querying huge result sets. Use OFFSET for pagination.

---

## Query Analysis

### Use SOQL Profiler (Developer Console)

1. Developer Console → Query Editor
2. Paste SOQL
3. Click Profile → Fetches count and time
4. If slow, add WHERE clause or check index

### Optimize Flow

1. **Count first**: How many records match?
2. **Check index**: Is WHERE using an indexed field?
3. **Paginate**: If 50K+ results, paginate
4. **Cache**: If same query runs often, cache result

---

## Checklist

- ✅ No SELECT *
- ✅ All queries have WHERE clause
- ✅ WHERE uses indexed field
- ✅ No SOQL in loops
- ✅ Aggregates in SOQL, not code
- ✅ No inequality operators (use positive logic)
- ✅ No LIKE without wildcards
- ✅ No concatenation (use bind variables)
- ✅ Pagination for large results
- ✅ Bulk tested (200+ records)

