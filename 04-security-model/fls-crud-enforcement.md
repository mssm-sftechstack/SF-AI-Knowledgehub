# FLS and CRUD Enforcement in Apex

**Salesforce doesn't automatically enforce field or object permissions in Apex. You have to do it yourself. The Salesforce security review will fail if you don't.**

## The Two Main Options

| Approach | How it handles inaccessible fields | Best for |
|---|---|---|
| `WITH SECURITY_ENFORCED` | Throws `System.QueryException` | Read queries where you want explicit failure |
| `Security.stripInaccessible()` | Strips fields silently, returns partial data | Read queries where partial data is acceptable |
| `WITH USER_MODE` (API 56.0+) | Enforces FLS + CRUD at DB level, strips on read | Write operations, preferred for new code |

## WITH SECURITY_ENFORCED

Added to the end of a SOQL query. If any field in SELECT or WHERE is inaccessible to the running user, the query throws.

```apex
public static List<Account> getAccountsWithRevenue(Set<Id> ids) {
    return [
        SELECT Id, Name, AnnualRevenue, BillingCountry
        FROM Account
        WHERE Id IN :ids
        WITH SECURITY_ENFORCED
    ];
}
```

Use this in Selector classes for read-only queries. If a field is inaccessible, you want to know. Silent data stripping leads to logic bugs downstream.

## Security.stripInaccessible()

Runs the query without `WITH SECURITY_ENFORCED`, then strips any fields the user can't access before you use the results.

```apex
public static List<Account> getAccountsSafe(Set<Id> ids) {
    List<Account> rawResults = [
        SELECT Id, Name, AnnualRevenue, BillingCountry
        FROM Account
        WHERE Id IN :ids
    ];

    SObjectAccessDecision decision = Security.stripInaccessible(
        AccessType.READABLE,
        rawResults
    );

    return (List<Account>) decision.getRecords();
}
```

Use this when you want to return partial data to the caller rather than throw. Common in UI-facing service methods where a user might not have access to all fields but should still see the record.

## WITH USER_MODE

Available from API 56.0 (Summer '23). Applies full FLS and CRUD enforcement at the database level for both SOQL and DML.

```apex
// SOQL with USER_MODE — strips inaccessible fields, enforces CRUD
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WHERE Id IN :ids
    WITH USER_MODE
];

// DML with USER_MODE
update as user accounts;
insert as user newContacts;
```

`WITH USER_MODE` is the preferred approach for new code. It covers both read and write in one consistent pattern. It throws on CRUD violations (can't read the object at all) and strips inaccessible fields on read.

## CRUD Checks for DML

Before DML operations, check object-level permissions explicitly:

```apex
public static void createAccounts(List<Account> accounts) {
    if (!Schema.sObjectType.Account.isCreateable()) {
        throw new SecurityException('Insufficient access to create Account records');
    }
    insert accounts;
}

public static void updateAccounts(List<Account> accounts) {
    if (!Schema.sObjectType.Account.isUpdateable()) {
        throw new SecurityException('Insufficient access to update Account records');
    }
    update accounts;
}

public static void deleteAccounts(List<Account> accounts) {
    if (!Schema.sObjectType.Account.isDeletable()) {
        throw new SecurityException('Insufficient access to delete Account records');
    }
    delete accounts;
}
```

Common CRUD check methods:

| Method | Checks |
|---|---|
| `isAccessible()` | Can the user query this object? |
| `isCreateable()` | Can the user insert records? |
| `isUpdateable()` | Can the user edit records? |
| `isDeletable()` | Can the user delete records? |

## Exempt Types

**Custom Metadata Types** (`__mdt`) and **Hierarchy Custom Settings** (`__c` with type Hierarchy) are exempt from FLS enforcement. You don't need `WITH SECURITY_ENFORCED` or `stripInaccessible()` for them. They're configuration data, not user data, and all users can read them.

```apex
// No SECURITY_ENFORCED needed for Custom Metadata Types
List<FeatureFlag__mdt> flags = [SELECT DeveloperName, IsEnabled__c FROM FeatureFlag__mdt];
```

## The runAs Test Trap

Tests run as the System Administrator by default. System Admin bypasses FLS. If your service uses `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()`, a test running as SysAdmin will never hit a field access violation.

Use `System.runAs()` with a standard user who has only the feature's Permission Set assigned:

```apex
@IsTest
static void testGetAccountsRespectsFieldAccess() {
    // Create a standard user with only the feature PS
    User testUser = TestUserFactory.createStandardUser();
    PermissionSet ps = [SELECT Id FROM PermissionSet WHERE Name = 'Account_Viewer' LIMIT 1];
    insert new PermissionSetAssignment(AssigneeId = testUser.Id, PermissionSetId = ps.Id);

    System.runAs(testUser) {
        List<Account> results = AccountSelector.getAccountsWithRevenue(
            new Set<Id>{ testAccount.Id }
        );
        // assert that AnnualRevenue is accessible / inaccessible as expected
    }
}
```

Create a shared `TestUserFactory` class used by all test classes that test FLS-enforcing services. Don't recreate the user setup in every test class.

```apex
@IsTest
public class TestUserFactory {
    public static User createStandardUser() {
        Profile p = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
        return new User(
            FirstName = 'Test',
            LastName = 'User',
            Email = 'testuser@example.com',
            Username = 'testuser_' + DateTime.now().getTime() + '@example.com',
            Alias = 'tuser',
            TimeZoneSidKey = 'America/Los_Angeles',
            LocaleSidKey = 'en_US',
            EmailEncodingKey = 'UTF-8',
            LanguageLocaleKey = 'en_US',
            ProfileId = p.Id
        );
    }
}
```
