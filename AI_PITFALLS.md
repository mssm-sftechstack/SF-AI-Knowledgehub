# AI Pitfalls in Salesforce Development

What AI tools commonly get wrong when writing Salesforce code. This document helps you catch mistakes before code review.

---

## The Core Problem

AI tools are trained on code from GitHub, Stack Overflow, and other public sources. Most of that code is:

- From dev/test environments (no production constraints)
- Written by developers learning (includes anti-patterns)
- Missing context about Salesforce specifics
- Not tested at scale (200+ records)

Result: AI writes code that looks correct but breaks in production.

---

## Pitfall 1: SOQL/DML in Loops (Query Limits)

### What AI Writes

```apex
public class AccountService {
  public static void updateAllAccountsWithDetails(List<Account> accounts) {
    for (Account acc : accounts) {
      List<Opportunity> opps = [SELECT Id, Amount FROM Opportunity WHERE AccountId = :acc.Id];
      
      Decimal total = 0;
      for (Opportunity opp : opps) {
        total += opp.Amount;
      }
      
      acc.TotalOpportunityAmount__c = total;
    }
    
    update accounts;
  }
}
```

**Problem**: If you pass 200 accounts, this runs 200 SOQL queries. Limit is 100.

### What You Should Write

```apex
public class AccountService {
  public static void updateAllAccountsWithDetails(List<Account> accounts) {
    // Single query: get all opportunities for all accounts
    List<Opportunity> allOpps = [
      SELECT AccountId, Amount FROM Opportunity 
      WHERE AccountId IN :new List<Id>(new Map<Id, Account>(accounts).keySet())
    ];
    
    // Group by account
    Map<Id, List<Opportunity>> oppsByAccount = new Map<Id, List<Opportunity>>();
    for (Opportunity opp : allOpps) {
      if (!oppsByAccount.containsKey(opp.AccountId)) {
        oppsByAccount.put(opp.AccountId, new List<Opportunity>());
      }
      oppsByAccount.get(opp.AccountId).add(opp);
    }
    
    // Update accounts
    for (Account acc : accounts) {
      Decimal total = 0;
      for (Opportunity opp : oppsByAccount.get(acc.Id) ?? new List<Opportunity>()) {
        total += opp.Amount;
      }
      acc.TotalOpportunityAmount__c = total;
    }
    
    update accounts;
  }
}
```

**Rule**: Zero SOQL inside loops. Query once, process in memory.

---

## Pitfall 2: Missing CRUD / FLS Checks

### What AI Writes

```apex
public class AccountController {
  @AuraEnabled(cacheable=true)
  public static List<Account> getAccounts() {
    return [SELECT Id, Name, Revenue__c, CustomField__c FROM Account];
  }
  
  @AuraEnabled
  public static void updateAccount(Account acc) {
    update acc;
  }
}
```

**Problem**: No CRUD checks. If user lacks read on `CustomField__c`, query still returns it (FLS violated).

### What You Should Write

```apex
public with sharing class AccountController {
  @AuraEnabled(cacheable=true)
  public static List<Account> getAccounts() {
    List<Account> accounts = [
      SELECT Id, Name, Revenue__c, CustomField__c FROM Account
    ];
    
    // Strip fields user can't read
    accounts = (List<Account>) Security.stripInaccessible(
      AccessType.READABLE,
      accounts
    ).getResults();
    
    return accounts;
  }
  
  @AuraEnabled
  public static void updateAccount(Account acc) {
    // Check update permission
    if (!Schema.Account.isUpdateable()) {
      throw new AuraHandledException('No update permission on Account');
    }
    
    // Strip fields user can't update
    acc = (Account) Security.stripInaccessible(
      AccessType.UPDATEABLE,
      acc
    ).getResults().get(0);
    
    update acc;
  }
}
```

**Rule**: Always check CRUD. Use stripInaccessible or WITH SECURITY_ENFORCED.

---

## Pitfall 3: No Error Handling

### What AI Writes

```javascript
// LWC
async handleSave() {
  const result = await saveAccount({ name: this.accountName });
  this.message = 'Saved successfully';
}
```

**Problem**: If Apex throws exception, no error handling. User sees nothing.

### What You Should Write

```javascript
async handleSave() {
  try {
    this.isLoading = true;
    const result = await saveAccount({ name: this.accountName });
    
    this.dispatchEvent(
      new ShowToastEvent({
        title: 'Success',
        message: 'Account saved',
        variant: 'success'
      })
    );
  } catch (error) {
    console.error('Error saving account:', error);
    
    this.dispatchEvent(
      new ShowToastEvent({
        title: 'Error',
        message: error.body?.message || 'Unknown error',
        variant: 'error'
      })
    );
  } finally {
    this.isLoading = false;
  }
}
```

**Rule**: All Apex calls need try-catch. All wire calls need error handling.

---

## Pitfall 4: Hardcoded IDs

### What AI Writes

```apex
public class OpportunityService {
  private static final Id OPPORTUNITY_RECORD_TYPE = '012a0000000IZ3AAM';
  
  public static void createOpportunity(String name) {
    Opportunity opp = new Opportunity(
      Name = name,
      RecordTypeId = OPPORTUNITY_RECORD_TYPE
    );
    insert opp;
  }
}
```

**Problem**: Record Type ID is specific to one org. Breaks when deployed to another org.

### What You Should Write

```apex
public class OpportunityService {
  private static final String RECORD_TYPE_NAME = 'Standard';
  
  public static void createOpportunity(String name) {
    Map<String, RecordTypeInfo> rtByName = Schema.SObjectType.Opportunity.getRecordTypeInfosByDeveloperName();
    
    if (!rtByName.containsKey(RECORD_TYPE_NAME)) {
      throw new ConfigException('Record Type ' + RECORD_TYPE_NAME + ' not found');
    }
    
    Id recordTypeId = rtByName.get(RECORD_TYPE_NAME).getRecordTypeId();
    
    Opportunity opp = new Opportunity(
      Name = name,
      RecordTypeId = recordTypeId
    );
    insert opp;
  }
}
```

**Rule**: Resolve IDs dynamically. Never hardcode Record Type IDs, Custom Setting IDs, or picklist values.

---

## Pitfall 5: No Bulk Testing

### What AI Writes

```apex
@IsTest
static void testCreateAccounts() {
  Account acc = new Account(Name = 'Test');
  insert acc;
  
  System.assertEquals('Test', [SELECT Name FROM Account WHERE Id = :acc.Id].Name);
}
```

**Problem**: Tests with 1 record. Code breaks with 200+ records due to SOQL limits.

### What You Should Write

```apex
@IsTest
static void testCreateAccountsBulk() {
  // Test with 200 records
  List<Account> accounts = new List<Account>();
  for (Integer i = 0; i < 200; i++) {
    accounts.add(new Account(Name = 'Account ' + i));
  }
  insert accounts;
  
  // Trigger logic runs on 200 records
  System.assertEquals(200, [SELECT COUNT() FROM Account]);
  
  // Assert within governor limits
  System.assert(Limits.getQueries() < 100, 'Exceeded query limit');
  System.assert(Limits.getDmlRows() <= 200, 'Exceeded DML limit');
}
```

**Rule**: Always bulk test. Code that works for 50 breaks at 200+.

---

## Pitfall 6: Not Using with sharing

### What AI Writes

```apex
public class AccountService {
  public static List<Account> getMyAccounts() {
    return [SELECT Id, Name FROM Account];
  }
}
```

**Problem**: No `with sharing` clause. Class ignores user's sharing rules. They see accounts they shouldn't.

### What You Should Write

```apex
public with sharing class AccountService {
  public static List<Account> getMyAccounts() {
    return [SELECT Id, Name FROM Account];
  }
}
```

**Rule**: All classes use `with sharing` by default. Only use `without sharing` for specific admin functions, with documentation.

---

## Pitfall 7: Forgetting Trigger Only Runs Once

### What AI Writes

```apex
trigger AccountTrigger on Account (after insert) {
  for (Account acc : Trigger.new) {
    Task task = new Task(WhoId = acc.OwnerId, Subject = 'Follow up');
    insert task;  // Creates 1 task per account
  }
}

// Test:
Account acc1 = new Account(Name = 'Acme', OwnerId = user1);
Account acc2 = new Account(Name = 'Tech', OwnerId = user2);
insert acc1;  // Trigger fires
insert acc2;  // Trigger fires again
```

**Problem**: Trigger fires twice (once per insert). If test inserts 2 records separately, 2 inserts = 2 DML statements. But trigger should batch.

### What You Should Write

```apex
trigger AccountTrigger on Account (after insert) {
  List<Task> tasks = new List<Task>();
  
  for (Account acc : Trigger.new) {
    tasks.add(new Task(WhoId = acc.OwnerId, Subject = 'Follow up'));
  }
  
  insert tasks;  // Batch insert outside loop
}

// Test:
List<Account> accounts = new List<Account>{
  new Account(Name = 'Acme', OwnerId = user1),
  new Account(Name = 'Tech', OwnerId = user2)
};
insert accounts;  // Trigger fires ONCE
```

**Rule**: Batch DML outside loops. Collect, then insert once.

---

## Pitfall 8: XSS in LWC (innerHTML)

### What AI Writes

```javascript
// LWC
export default class UserBio extends LightningElement {
  @track bio;
  
  connectedCallback() {
    getUserBio().then(result => {
      this.bio = result;
    });
  }
}

// Template
<template>
  <div innerHTML={bio}></div>
</template>
```

**Problem**: If `bio` contains `<img onerror="alert('xss')">`, it executes. XSS vulnerability.

### What You Should Write

```javascript
// Template
<template>
  <div>{bio}</div>  <!-- Auto-escaped, safe -->
</template>

// Or if you must use HTML:
import DOMPurify from 'isomorphic-dompurify';

connectedCallback() {
  getUserBio().then(result => {
    this.bio = DOMPurify.sanitize(result);  // Sanitize before use
  });
}
```

**Rule**: Never use innerHTML with user input. Use text binding (auto-escaped) or sanitize.

---

## Pitfall 9: No Callout from Trigger

### What AI Writes

```apex
trigger AccountTrigger on Account (after insert) {
  for (Account acc : Trigger.new) {
    callExternalAPI(acc);  // ❌ Callout in trigger context
  }
}
```

**Problem**: Callouts can't run in trigger context. Throws exception at runtime.

### What You Should Write

```apex
trigger AccountTrigger on Account (after insert) {
  System.enqueueJob(new UpdateExternalSystemQueueable(Trigger.new));
}

public class UpdateExternalSystemQueueable implements Queueable, Database.AllowsCallouts {
  private List<Account> accounts;
  
  public UpdateExternalSystemQueueable(List<Account> accounts) {
    this.accounts = accounts;
  }
  
  public void execute(QueueableContext context) {
    for (Account acc : accounts) {
      callExternalAPI(acc);  // ✅ Callout in Queueable
    }
  }
}
```

**Rule**: Never callout from trigger. Use Queueable with Database.AllowsCallouts.

---

## Pitfall 10: Wire Adapter Without Error Handling

### What AI Writes

```javascript
@wire(getAccounts)
wiredAccounts(data) {
  this.accounts = data;  // No error handling
}
```

**Problem**: If Apex fails, `data` is undefined. Component breaks silently.

### What You Should Write

```javascript
@wire(getAccounts)
wiredAccounts({ data, error }) {
  if (data) {
    this.accounts = data;
    this.error = undefined;
  } else if (error) {
    this.error = error.body?.message || 'Unknown error';
    this.accounts = undefined;
  }
}
```

**Rule**: Wire adapters must handle both `data` and `error`.

---

## Pitfall 11: Missing Test Coverage for Error Paths

### What AI Writes

```apex
@IsTest
static void testDeleteAccount() {
  Account acc = new Account(Name = 'Test');
  insert acc;
  delete acc;
  
  System.assertEquals(0, [SELECT COUNT() FROM Account WHERE Name = 'Test']);
}
```

**Problem**: Only tests happy path. Doesn't test what happens if delete fails (e.g., record locked).

### What You Should Write

```apex
@IsTest
static void testDeleteAccount() {
  Account acc = new Account(Name = 'Test');
  insert acc;
  delete acc;
  
  System.assertEquals(0, [SELECT COUNT() FROM Account WHERE Name = 'Test']);
}

@IsTest
static void testDeleteAccountWithoutPermission() {
  User limitedUser = [SELECT Id FROM User WHERE Email = 'limited@example.com'];
  Account acc = [SELECT Id FROM Account LIMIT 1];
  
  System.runAs(limitedUser) {
    try {
      delete acc;
      System.assert(false, 'Should fail without permission');
    } catch (DmlException ex) {
      System.assert(ex.getMessage().contains('INSUFFICIENT_ACCESS'));
    }
  }
}
```

**Rule**: Test error paths. Test permissions. Test edge cases.

---

## Pitfall 12: Not Handling Null in Collections

### What AI Writes

```apex
public List<String> getAccountNames(List<Account> accounts) {
  List<String> names = new List<String>();
  
  for (Account acc : accounts) {
    names.add(acc.Name);  // What if Name is null?
  }
  
  return names;
}
```

**Problem**: If account.Name is null, adds null to list. Breaks formatting downstream.

### What You Should Write

```apex
public List<String> getAccountNames(List<Account> accounts) {
  List<String> names = new List<String>();
  
  for (Account acc : accounts) {
    if (acc.Name != null) {
      names.add(acc.Name);
    }
  }
  
  return names;
}
```

**Rule**: Check for null before using values.

---

## Checklist: Catching AI Mistakes

Before code review:

- ✅ No SOQL/DML in loops
- ✅ CRUD/FLS checks present
- ✅ Error handling in all Apex calls and wire adapters
- ✅ No hardcoded IDs (resolve dynamically)
- ✅ Bulk tested (200+ records)
- ✅ All classes use `with sharing`
- ✅ No callouts in trigger context
- ✅ No innerHTML without sanitization
- ✅ No null pointer exceptions (check before use)
- ✅ Test coverage includes error paths
- ✅ Test includes permission testing (System.runAs)

