# Service and Selector Pattern

**SOQL belongs in one place. DML belongs in one place. Scatter them and debugging becomes archaeology.**

---

## The Problem With Scattered Queries

```apex
// OpportunityTriggerHandler.cls
List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];

// AccountService.cls
List<Account> accounts = [SELECT Id, Name, Industry FROM Account WHERE Id IN :accountIds];

// CaseService.cls
List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds WITH SECURITY_ENFORCED];
```

Three queries on the same object. One is missing `SECURITY_ENFORCED`. They return different fields. When Account gets a new security requirement, you update two of the three and miss the third.

---

## The Selector Class

All SOQL lives here. Nothing else calls the database directly.

```apex
public inherited sharing class AccountSelector {

    public List<Account> getByIds(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Industry, Rating, OwnerId, BillingCity, BillingCountry
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
            ORDER BY Name ASC
        ];
    }

    public List<Account> getWithOpenCases(Set<Id> accountIds) {
        return [
            SELECT Id, Name,
                (SELECT Id, Subject, Status FROM Cases WHERE Status != 'Closed')
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
        ];
    }
}
```

`inherited sharing` on the selector: sharing enforcement comes from the caller, not the selector itself. The same selector works in both `with sharing` and `without sharing` contexts.

---

## The Service Class

Business logic lives here. It calls the selector for data, applies rules, and returns results.

```apex
public with sharing class AccountService {

    private AccountSelector selector;

    public AccountService() {
        this.selector = new AccountSelector();
    }

    // constructor for test injection
    @TestVisible
    private AccountService(AccountSelector selector) {
        this.selector = selector;
    }

    public void assignDefaultRating(List<Account> accounts) {
        for (Account acc : accounts) {
            if (acc.Rating == null && acc.Industry == 'Technology') {
                acc.Rating = 'Hot';
            }
        }
    }

    public Map<Id, List<Case>> getOpenCasesByAccount(Set<Id> accountIds) {
        List<Account> accounts = selector.getWithOpenCases(accountIds);
        Map<Id, List<Case>> result = new Map<Id, List<Case>>();
        for (Account acc : accounts) {
            result.put(acc.Id, acc.Cases);
        }
        return result;
    }
}
```

---

## Why This Structure Works

| Benefit | Detail |
|---|---|
| Testability | Inject a mock selector in tests, no database needed |
| Security | `WITH SECURITY_ENFORCED` lives in one file per object |
| Performance | Spot duplicate queries immediately, centralize optimisation |
| Maintainability | Add a field to one query, all callers get it |

---

## Static vs Instance Selectors

Instance wins for testability. Static methods can't be overridden. If your service uses a static selector, you can't inject a mock.

**Wrong:**
```apex
public class AccountService {
    public static List<Account> enrichAccounts(Set<Id> ids) {
        List<Account> accounts = AccountSelector.getByIds(ids); // static call, can't mock
        // ...
    }
}
```

**Right:**
```apex
public with sharing class AccountService {
    private AccountSelector selector;

    public AccountService() {
        this.selector = new AccountSelector();
    }

    @TestVisible
    private AccountService(AccountSelector selectorOverride) {
        this.selector = selectorOverride;
    }
}
```

In tests:

```apex
@IsTest
static void testEnrichAccounts() {
    // arrange
    AccountSelector mockSelector = new MockAccountSelector(); // test double
    AccountService service = new AccountService(mockSelector);

    // act + assert
    // no database calls, runs in milliseconds
}
```

---

## Complete Working Example

```apex
// AccountSelector.cls
public inherited sharing class AccountSelector {

    public List<Account> getByIds(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Industry, Rating, AnnualRevenue
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
        ];
    }

    public List<Account> getByIndustry(String industry) {
        return [
            SELECT Id, Name, Industry, Rating
            FROM Account
            WHERE Industry = :industry
            WITH SECURITY_ENFORCED
            ORDER BY Name ASC
            LIMIT 1000
        ];
    }
}

// AccountService.cls
public with sharing class AccountService {

    private AccountSelector selector;

    public AccountService() {
        this.selector = new AccountSelector();
    }

    @TestVisible
    private AccountService(AccountSelector sel) {
        this.selector = sel;
    }

    public void setRatingByRevenue(List<Account> accounts) {
        for (Account acc : accounts) {
            if (acc.AnnualRevenue == null) continue;
            if (acc.AnnualRevenue >= 10000000) {
                acc.Rating = 'Hot';
            } else if (acc.AnnualRevenue >= 1000000) {
                acc.Rating = 'Warm';
            } else {
                acc.Rating = 'Cold';
            }
        }
    }

    public List<Account> getEnrichedAccounts(Set<Id> accountIds) {
        List<Account> accounts = selector.getByIds(accountIds);
        setRatingByRevenue(accounts);
        return accounts;
    }
}
```
