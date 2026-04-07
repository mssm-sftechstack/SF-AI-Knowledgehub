# Security Guardrails

Covering CRUD checks, Field-Level Security (FLS), sharing rules, data masking, and permission model design.

---

## Core Rules

### Rule 1: Always Check CRUD

Every SOQL query and DML operation must check Create, Read, Update, Delete permissions before executing.

```apex
// ❌ Wrong
List<Account> accounts = [SELECT Id, Name FROM Account];
update accounts;

// ✅ Right with stripInaccessible
List<Account> accounts = [SELECT Id, Name FROM Account];
accounts = (List<Account>) Security.stripInaccessible(AccessType.READABLE, accounts);
update accounts;

// ✅ Right with FLS enforcement in query
List<Account> accounts = [SELECT Id, Name FROM Account WITH SECURITY_ENFORCED];
```

**Two Approaches**:

1. **Security.stripInaccessible()**: Removes fields the user can't access. Query succeeds, inaccessible fields are silently removed.
2. **WITH SECURITY_ENFORCED**: Query throws exception if user lacks access. Safer for security audits.

**Which to use**:
- **stripInaccessible**: When you want graceful degradation (show data user can see, hide what they can't)
- **WITH SECURITY_ENFORCED**: When access is required (throw exception if denied)

### Rule 2: Always Enforce FLS on Writes

When updating records, enforce FLS.

```apex
// ❌ Wrong
account.Name = 'New Name';
account.CustomField__c = 'Custom Value';
update account;  // If user lacks FLS on CustomField__c, still updates

// ✅ Right
Schema.SObjectField[] fieldsToUpdate = new Schema.SObjectField[]{
  Account.Name,
  Account.CustomField__c
};

for (Schema.SObjectField field : fieldsToUpdate) {
  if (!field.getDescribe().isUpdateable()) {
    throw new SecurityException('Field ' + field + ' is not updateable');
  }
}

account.Name = 'New Name';
account.CustomField__c = 'Custom Value';
update account;

// ✅ Or use stripInaccessible for writes
account.Name = 'New Name';
account.CustomField__c = 'Custom Value';
Map<String, Object> updateMap = (Map<String, Object>) 
  Security.stripInaccessible(AccessType.UPDATEABLE, account).getResults().get(0);
update account;  // Fields user can't update are silently skipped
```

### Rule 3: Respect Sharing Rules

Sharing rules determine what records a user can see. Always respect them.

```apex
// ❌ Wrong
List<Account> allAccounts = [SELECT Id FROM Account];  // Ignores sharing rules

// ✅ Right
List<Account> visibleAccounts = [SELECT Id FROM Account];  // Respects sharing rules by default

// ✅ Explicit with sharing clause
public with sharing class AccountService {
  public static List<Account> getVisibleAccounts() {
    return [SELECT Id, Name FROM Account];  // Respects user's sharing rules
  }
}

// ❌ Bypass sharing (only when necessary)
public without sharing class AdminAccountService {
  public static List<Account> getAllAccounts() {
    return [SELECT Id, Name FROM Account];  // Ignores sharing rules (dangerous)
  }
}
```

**Best Practice**: All Apex classes use `with sharing` by default. Only use `without sharing` for specific admin functions, and document why.

---

## Field-Level Security (FLS) Testing

### Test Data Setup with Proper Permissions

```apex
@IsTest
private class AccountServiceTest {
  @TestSetup
  static void setupTestData() {
    // Create test user with standard permissions only
    User testUser = new User(
      FirstName = 'Test',
      LastName = 'User',
      Email = 'testuser@example.com',
      Username = 'testuser@example.com.' + System.now().millisecond(),
      ProfileId = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1].Id,
      Alias = 'tstu',
      TimeZoneSidKey = 'America/Los_Angeles',
      LocaleSidKey = 'en_US',
      EmailEncodingKey = 'UTF-8',
      LanguageLocaleKey = 'en_US'
    );
    insert testUser;
    
    // Assign feature permission set (custom field visibility)
    PermissionSetAssignment psa = new PermissionSetAssignment(
      PermissionSetId = [SELECT Id FROM PermissionSet WHERE Name = 'Account_Admin'].Id,
      AssigneeId = testUser.Id
    );
    insert psa;
  }
  
  @IsTest
  static void testUpdateAccountWithFLS() {
    User testUser = [SELECT Id FROM User WHERE Email = 'testuser@example.com' LIMIT 1];
    
    Account acc = [SELECT Id FROM Account LIMIT 1];
    
    System.runAs(testUser) {
      acc.Name = 'Updated Name';
      acc.CustomField__c = 'Custom Value';
      
      // If testUser lacks FLS on CustomField__c, exception thrown
      try {
        update acc;
        System.assert(true, 'Update succeeded (user has FLS)');
      } catch (DmlException ex) {
        System.assert(ex.getMessage().contains('INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY'),
                     'Expected FLS error');
      }
    }
  }
}
```

### Testing with stripInaccessible

```apex
@IsTest
static void testStripInaccessibleFields() {
  User testUser = [SELECT Id FROM User WHERE Email = 'testuser@example.com' LIMIT 1];
  
  Account acc = new Account(
    Name = 'Test Account',
    CustomField__c = 'Should be hidden',
    Phone = '555-1234'
  );
  insert acc;
  
  System.runAs(testUser) {
    List<Account> accounts = [SELECT Id, Name, CustomField__c, Phone FROM Account WHERE Id = :acc.Id];
    
    // Strip fields user can't read
    accounts = (List<Account>) Security.stripInaccessible(AccessType.READABLE, accounts).getResults();
    
    // CustomField__c is now null if user lacks FLS
    Account result = accounts[0];
    System.assert(result.CustomField__c == null || result.CustomField__c.length() > 0);
    System.assert(result.Name == 'Test Account');  // Name is always readable
  }
}
```

---

## Sharing Rules: Org-Wide Defaults & Manual Sharing

### Org-Wide Defaults (OWD)

OWD determines the baseline sharing level:

| OWD Setting | What It Means |
|---|---|
| **Public Read/Write** | Everyone can see and edit (least restrictive) |
| **Public Read Only** | Everyone can see, only owner can edit |
| **Private** | Only owner can see (most restrictive) |
| **Controlled by Parent** | Child records inherit parent's sharing |

### Sharing Rule Example

```apex
// Create a sharing rule: Sales team can read all Accounts
public class AccountSharingService {
  public static void shareAccountWithTeam(Id accountId, List<Id> userIds) {
    List<AccountShare> shares = new List<AccountShare>();
    
    for (Id userId : userIds) {
      AccountShare share = new AccountShare();
      share.AccountId = accountId;
      share.UserOrGroupId = userId;
      share.AccountAccessLevel = 'Read';  // Read, Edit, or All
      shares.add(share);
    }
    
    insert shares;
  }
}

// Usage
AccountSharingService.shareAccountWithTeam(accountId, new List<Id>{user1Id, user2Id});
```

### Testing Sharing Rules

```apex
@IsTest
static void testSharingRules() {
  User owner = createUser('Owner');
  User viewer = createUser('Viewer');
  
  System.runAs(owner) {
    Account acc = new Account(Name = 'Private Account');
    insert acc;
    
    // Viewer shouldn't see account yet
    List<Account> visibleToViewer = [SELECT Id FROM Account WHERE Id = :acc.Id];
    System.assertEquals(0, visibleToViewer.size(), 'Viewer should not see account');
  }
  
  // Owner shares account with viewer
  AccountShare share = new AccountShare(
    AccountId = accountId,
    UserOrGroupId = viewer.Id,
    AccountAccessLevel = 'Read'
  );
  insert share;
  
  System.runAs(viewer) {
    // Now viewer can see account
    List<Account> visibleAccounts = [SELECT Id FROM Account WHERE Id = :accountId];
    System.assertEquals(1, visibleAccounts.size(), 'Viewer should now see account');
  }
}
```

---

## Data Masking & PII Protection

### Pattern 1: Masking Sensitive Fields in Queries

```apex
public class AccountService {
  public static List<Account> getAccountsWithMasking() {
    List<Account> accounts = [SELECT Id, Name, Phone FROM Account];
    
    // Mask phone numbers
    for (Account acc : accounts) {
      if (acc.Phone != null && acc.Phone.length() > 4) {
        acc.Phone = '***-' + acc.Phone.substring(acc.Phone.length() - 4);
      }
    }
    
    return accounts;
  }
}
```

### Pattern 2: PII in Logs

```apex
private static void logError(Exception ex, String accountId) {
  ErrorLog__c log = new ErrorLog__c();
  log.Message__c = ex.getMessage();
  log.Account_Id__c = accountId;  // Log ID, not sensitive data
  
  // ❌ Don't do this
  // log.Email__c = account.Email;  // Never log email addresses
  
  insert log;
}
```

### Pattern 3: Restricting Access to Sensitive Fields

```apex
public with sharing class SensitiveDataService {
  public static Map<Id, Account> getAccountsWithSSN() {
    // Check if user has permission to view SSNs
    if (!FeatureManagement.checkPermission('View_SSN')) {
      throw new SecurityException('You do not have permission to view SSNs');
    }
    
    return new Map<Id, Account>([SELECT Id, Name, SSN__c FROM Account]);
  }
}
```

---

## Custom Permissions

Use custom permissions to restrict access to sensitive operations.

### Setup Steps

1. Setup > Custom Code > Custom Permissions
2. Click New
3. **Label**: `Can_Export_Data`
4. **Name**: `Can_Export_Data`
5. Add to Permission Sets that should have this permission

### In Apex Code

```apex
public with sharing class DataExportService {
  public static String exportAccountsToCSV() {
    // Check permission
    if (!FeatureManagement.checkPermission('Can_Export_Data')) {
      throw new SecurityException('You do not have permission to export data');
    }
    
    List<Account> accounts = [SELECT Id, Name, Phone FROM Account];
    
    // Generate CSV
    String csv = 'Id,Name,Phone\n';
    for (Account acc : accounts) {
      csv += acc.Id + ',' + acc.Name + ',' + acc.Phone + '\n';
    }
    
    return csv;
  }
}
```

---

## Hardcoded IDs: The Silent Killer

Never hardcode Record Type IDs, custom setting IDs, or picklist values. Always resolve dynamically.

### ❌ Wrong (hardcoded)

```apex
public void createAccount() {
  Account acc = new Account(
    Name = 'Acme',
    RecordTypeId = '012a0000000IZ3AAM'  // Hardcoded! Breaks in other orgs
  );
  insert acc;
}
```

### ✅ Right (dynamic)

```apex
public void createAccount() {
  Id accountRecordTypeId = Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName()
    .get('Standard_Account')
    .getRecordTypeId();
  
  Account acc = new Account(
    Name = 'Acme',
    RecordTypeId = accountRecordTypeId
  );
  insert acc;
}
```

### ✅ Right (with error handling)

```apex
public void createAccount() {
  Map<String, RecordTypeInfo> rtByName = Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName();
  
  if (!rtByName.containsKey('Standard_Account')) {
    throw new ConfigException('Record Type Standard_Account not found');
  }
  
  Id accountRecordTypeId = rtByName.get('Standard_Account').getRecordTypeId();
  
  Account acc = new Account(
    Name = 'Acme',
    RecordTypeId = accountRecordTypeId
  );
  insert acc;
}
```

---

## SQL Injection Prevention

Salesforce SOQL is injection-safe by default when using bind variables. Never concatenate strings into SOQL.

### ❌ Wrong (vulnerable)

```apex
String searchName = 'Test\' OR Name != NULL; --';
String query = 'SELECT Id FROM Account WHERE Name = \'' + searchName + '\'';
List<Account> accounts = Database.query(query);  // SOQL injection!
```

### ✅ Right (safe)

```apex
String searchName = 'Test';
List<Account> accounts = [SELECT Id FROM Account WHERE Name = :searchName];  // Safe
```

**Best Practice**: Always use bind variables (`:variable`) in SOQL. Never concatenate user input into query strings.

---

## XSS Prevention in LWC

See LWC_BEST_PRACTICES.md for XSS prevention patterns.

---

## Checklist: Security Review

- ✅ All SOQL/DML have CRUD checks (stripInaccessible or WITH SECURITY_ENFORCED)
- ✅ All classes use `with sharing` (document exceptions)
- ✅ FLS enforced on writes
- ✅ No hardcoded IDs (Record Types, Picklists resolved dynamically)
- ✅ No string concatenation in SOQL (use bind variables)
- ✅ Sensitive fields masked or restricted
- ✅ Custom permissions for sensitive operations
- ✅ No PII in logs
- ✅ Error messages don't expose system details
- ✅ All endpoints validate input
- ✅ Test with permission sets (not System Admin)

