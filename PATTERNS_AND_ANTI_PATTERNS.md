# Standardized Design Patterns (No Hallucinations)

**CRITICAL DIRECTIVE FOR AI:** Do not invent your own frameworks. Do not write custom trigger handlers from scratch. You must adhere to standard, industry-accepted Salesforce design patterns to ensure maintainability.

## 1. The Trigger Handler Pattern
NEVER write logic inside a `.trigger` file. A trigger should only contain one line of execution code.

```apex
// ✅ MANDATORY TRIGGER SKELETON
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    AccountTriggerHandler.run();
}
```

## 2. The Selector Pattern (SOQL Encapsulation)
Do not scatter SOQL queries throughout your services or controllers. If you are querying the same object in multiple places, use a Selector class.

```apex
// ✅ MANDATORY SELECTOR PATTERN
public with sharing class AccountSelector {
    public static List<Account> getActiveAccounts(Set<Id> accountIds) {
        return [SELECT Id, Name, Industry 
                FROM Account 
                WHERE Id IN :accountIds AND Status__c = 'Active' 
                WITH USER_MODE];
    }
}
```

## 3. The Callout Anti-Pattern
**Anti-Pattern:** Using a `try/catch` block for an HTTP callout but failing to log the response body on a failure.
* **The Rule:** If `res.getStatusCode() != 200`, you MUST log the `res.getBody()` to a custom log object or output it via `System.debug(LoggingLevel.ERROR)`. Do not swallow exceptions silently.