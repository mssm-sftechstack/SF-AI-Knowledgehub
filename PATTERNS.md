# Reusable Patterns

Copy, adapt, and use these patterns in your projects.

---

## Pattern 1: Trigger Handler with Service Layer

Use this for any trigger that needs business logic.

```apex
// Trigger (one per object)
trigger Account_Trigger on Account (before insert, before update, after insert, after update) {
  if (Trigger.isBefore) {
    if (Trigger.isInsert) {
      Account_Before_Insert_Handler.handle(Trigger.new);
    }
    if (Trigger.isUpdate) {
      Account_Before_Update_Handler.handle(Trigger.newMap, Trigger.oldMap);
    }
  }
  if (Trigger.isAfter) {
    if (Trigger.isInsert) {
      Account_After_Insert_Handler.handle(Trigger.newMap);
    }
    if (Trigger.isUpdate) {
      Account_After_Update_Handler.handle(Trigger.newMap, Trigger.oldMap);
    }
  }
}

// Handler (routes to service)
public with sharing class Account_After_Update_Handler {
  public static void handle(Map<Id, Account> newMap, Map<Id, Account> oldMap) {
    List<Account> changedAccounts = new List<Account>();
    
    for (Account newAcc : newMap.values()) {
      Account oldAcc = oldMap.get(newAcc.Id);
      if (newAcc.Status__c != oldAcc.Status__c) {
        changedAccounts.add(newAcc);
      }
    }
    
    if (!changedAccounts.isEmpty()) {
      Account_Service.onStatusChange(changedAccounts);
    }
  }
}

// Service (business logic)
public with sharing class Account_Service {
  public static void onStatusChange(List<Account> accounts) {
    // Your business logic here
  }
}
```

---

## Pattern 2: Async Work with Queueable

Use this for heavy processing or callouts.

```apex
public with sharing class Account_Sync_Queueable 
  implements Queueable, Database.AllowsCallouts {
  
  private List<Account> accounts;
  private Integer retryCount;
  
  public Account_Sync_Queueable(List<Account> accounts) {
    this.accounts = accounts;
    this.retryCount = 0;
  }
  
  public void execute(QueueableContext ctx) {
    try {
      for (Account acc : accounts) {
        ExternalSystem.sync(acc);
      }
    } catch (Exception e) {
      if (retryCount < 3) {
        retryCount++;
        System.enqueueJob(new Account_Sync_Queueable(accounts));
      } else {
        System.debug(LoggingLevel.ERROR, 'Sync failed after 3 retries: ' + e.getMessage());
      }
    }
  }
}

// Call from handler
System.enqueueJob(new Account_Sync_Queueable(changedAccounts));
```

---

## Pattern 3: Selector with WITH SECURITY_ENFORCED

Use this for all SOQL queries.

```apex
public with sharing class Account_Selector {
  
  public static List<Account> getActiveAccounts() {
    return [SELECT Id, Name, Phone, Email__c, Status__c 
            FROM Account 
            WHERE Status__c = 'Active'
            WITH SECURITY_ENFORCED
            LIMIT 50000];
  }
  
  public static Map<Id, Account> getAccountsByIds(Set<Id> accountIds) {
    return new Map<Id, Account>(
      [SELECT Id, Name, Phone, Status__c 
       FROM Account 
       WHERE Id IN :accountIds
       WITH SECURITY_ENFORCED]
    );
  }
  
  public static List<AggregateResult> getOpportunityCounts(Set<Id> accountIds) {
    return [SELECT AccountId, COUNT(Id) oppCount 
            FROM Opportunity 
            WHERE AccountId IN :accountIds
            GROUP BY AccountId
            WITH SECURITY_ENFORCED];
  }
}
```

---

## Pattern 4: Bulkified Update Logic

Collect IDs, query once, update once.

```apex
public with sharing class Account_Service {
  
  public static void updateOppCount(List<Account> accounts) {
    Set<Id> accountIds = new Set<Id>();
    for (Account acc : accounts) {
      accountIds.add(acc.Id);
    }
    
    // Query once
    Map<Id, List<Opportunity>> oppsByAccount = 
      getOpportunitiesByAccount(accountIds);
    
    // Iterate and update
    for (Account acc : accounts) {
      acc.Opp_Count__c = 
        oppsByAccount.containsKey(acc.Id) 
        ? oppsByAccount.get(acc.Id).size() 
        : 0;
    }
    
    // Update once
    update accounts;
  }
  
  private static Map<Id, List<Opportunity>> getOpportunitiesByAccount(
    Set<Id> accountIds) {
    
    Map<Id, List<Opportunity>> result = 
      new Map<Id, List<Opportunity>>();
    
    for (Opportunity opp : [SELECT AccountId FROM Opportunity 
                            WHERE AccountId IN :accountIds
                            WITH SECURITY_ENFORCED]) {
      if (!result.containsKey(opp.AccountId)) {
        result.put(opp.AccountId, new List<Opportunity>());
      }
      result.get(opp.AccountId).add(opp);
    }
    
    return result;
  }
}
```

---

## Pattern 5: Test Class with Bulk Testing

Tests that catch real problems.

```apex
@IsTest
private class Account_Service_Test {
  
  @TestSetup
  static void setup() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
      accounts.add(new Account(Name = 'Test Account ' + i));
    }
    insert accounts;
  }
  
  @IsTest
  static void testBulkUpdateOppCount() {
    List<Account> accounts = [SELECT Id FROM Account];
    
    List<Opportunity> opps = new List<Opportunity>();
    for (Account acc : accounts) {
      opps.add(new Opportunity(
        AccountId = acc.Id,
        Name = 'Test Opp',
        StageName = 'Prospecting',
        CloseDate = Date.today().addDays(30)
      ));
    }
    insert opps;
    
    Test.startTest();
    Account_Service.updateOppCount(accounts);
    Test.stopTest();
    
    List<Account> result = [SELECT Opp_Count__c FROM Account];
    for (Account acc : result) {
      Assert.isNotNull(acc.Opp_Count__c);
    }
  }
  
  @IsTest
  static void testWithFLS() {
    User standardUser = createStandardUser();
    
    System.runAs(standardUser) {
      List<Account> accounts = [SELECT Id, Name FROM Account];
      Assert.isNotNull(accounts);
    }
  }
  
  private static User createStandardUser() {
    User u = new User(
      FirstName = 'Test',
      LastName = 'User',
      Email = 'test@example.com',
      Username = 'testuser@example.com',
      Alias = 'testuser',
      TimeZoneSidKey = 'America/Los_Angeles',
      LocaleSidKey = 'en_US',
      EmailEncodingKey = 'UTF-8',
      ProfileId = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1].Id
    );
    insert u;
    return u;
  }
}
```

---

## Pattern 6: Service with Error Handling

Graceful error handling without being verbose.

```apex
public with sharing class Account_Service {
  
  public static void validateAccounts(List<Account> accounts) {
    for (Account acc : accounts) {
      if (String.isBlank(acc.Name)) {
        acc.addError('Account name is required');
      }
      
      if (acc.Industry__c == null) {
        acc.addError('Industry is required');
      }
    }
  }
  
  public static void syncToExternalSystem(List<Account> accounts) {
    try {
      for (Account acc : accounts) {
        ExternalSystem.sync(acc);
      }
    } catch (CalloutException e) {
      System.debug(LoggingLevel.ERROR, 'Callout failed: ' + e.getMessage());
      throw new AuraHandledException('External system is unavailable');
    } catch (Exception e) {
      System.debug(LoggingLevel.ERROR, 'Unexpected error: ' + e.getMessage());
      throw new AuraHandledException('An error occurred');
    }
  }
}
```

---

## Pattern 7: LWC Data Loading (LDS First)

Load record data with Lightning Data Service before Apex.

```javascript
import { LightningElement, api } from 'lwc';
import { getRecord } from 'lightning/uiRecordApi';

const FIELDS = ['Account.Id', 'Account.Name', 'Account.Phone'];

export default class AccountDetail extends LightningElement {
  @api recordId;
  account;
  loading = true;
  error;
  
  connectedCallback() {
    this.loadRecord();
  }
  
  loadRecord() {
    getRecord({ recordId: this.recordId, fields: FIELDS })
      .then(result => {
        this.account = result;
        this.error = undefined;
        this.loading = false;
      })
      .catch(error => {
        this.error = error;
        this.loading = false;
      });
  }
}
```

---

## Pattern 8: Flow Calling Apex Action

Flow calls Apex for complex logic.

```apex
public class Calculate_Service {
  
  @InvocableMethod(label='Calculate Revenue')
  public static List<Result> calculateRevenue(List<Request> requests) {
    List<Result> results = new List<Result>();
    
    for (Request req : requests) {
      Result res = new Result();
      res.revenue = req.amount * req.quantity;
      results.add(res);
    }
    
    return results;
  }
  
  public class Request {
    @InvocableVariable(label='Amount' required=true)
    public Decimal amount;
    
    @InvocableVariable(label='Quantity' required=true)
    public Integer quantity;
  }
  
  public class Result {
    @InvocableVariable(label='Revenue')
    public Decimal revenue;
  }
}
```

Then in Flow: Add an Action → Apex → Calculate_Revenue → Map inputs and outputs.
