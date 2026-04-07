# Test Data Patterns

Factory patterns, bulk testing, and permission testing.

---

## Test Data Factory Pattern

### Basic Factory

```apex
@IsTest
public class TestDataFactory {
  public static Account createAccount(String name) {
    return new Account(Name = name);
  }
  
  public static Contact createContact(String firstName, String lastName, Id accountId) {
    return new Contact(FirstName = firstName, LastName = lastName, AccountId = accountId);
  }
  
  public static Opportunity createOpportunity(String name, Id accountId, String stage) {
    return new Opportunity(
      Name = name,
      AccountId = accountId,
      StageName = stage,
      CloseDate = Date.today().addDays(30)
    );
  }
  
  // Bulk factory
  public static List<Account> createAccounts(Integer count) {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < count; i++) {
      accounts.add(new Account(Name = 'Account ' + i));
    }
    return accounts;
  }
}

// Usage
@IsTest
static void testCreateAccount() {
  Account acc = TestDataFactory.createAccount('Acme');
  insert acc;
  
  System.assertEquals('Acme', acc.Name);
}

@IsTest
static void testBulk() {
  List<Account> accounts = TestDataFactory.createAccounts(200);
  insert accounts;
  
  System.assertEquals(200, [SELECT COUNT() FROM Account]);
}
```

### Advanced Factory with Builder Pattern

```apex
@IsTest
public class TestDataFactory {
  public class AccountBuilder {
    private String name = 'Default Account';
    private String industry = 'Technology';
    private Decimal revenue = 1000000;
    
    public AccountBuilder withName(String name) {
      this.name = name;
      return this;
    }
    
    public AccountBuilder withIndustry(String industry) {
      this.industry = industry;
      return this;
    }
    
    public AccountBuilder withRevenue(Decimal revenue) {
      this.revenue = revenue;
      return this;
    }
    
    public Account build() {
      return new Account(Name = name, Industry = industry, Revenue__c = revenue);
    }
  }
}

// Usage
@IsTest
static void testCustomAccount() {
  Account acc = new TestDataFactory.AccountBuilder()
    .withName('Custom Corp')
    .withRevenue(5000000)
    .build();
  
  insert acc;
  
  System.assertEquals(5000000, acc.Revenue__c);
}
```

---

## @TestSetup vs. Test Method Data

### Pattern 1: @TestSetup (Shared Data)

```apex
@IsTest
private class AccountServiceTest {
  @TestSetup
  static void setupTestData() {
    // Data created once per class, shared across all test methods
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 10; i++) {
      accounts.add(new Account(Name = 'Account ' + i));
    }
    insert accounts;
  }
  
  @IsTest
  static void testGetAccounts() {
    List<Account> accounts = [SELECT Id FROM Account];
    System.assertEquals(10, accounts.size());
  }
  
  @IsTest
  static void testUpdateAccounts() {
    List<Account> accounts = [SELECT Id FROM Account];
    for (Account acc : accounts) {
      acc.Revenue__c = 50000;
    }
    update accounts;
    
    System.assertEquals(10, [SELECT COUNT() FROM Account WHERE Revenue__c = 50000]);
  }
}
```

**Pros**: Data created once, reused across methods, faster.

**Cons**: Tests are not fully isolated (modifications in one test affect another).

### Pattern 2: Test Method Data (Isolated)

```apex
@IsTest
private class AccountServiceTest {
  @IsTest
  static void testGetAccounts() {
    // Create data in test method
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 10; i++) {
      accounts.add(new Account(Name = 'Account ' + i));
    }
    insert accounts;
    
    System.assertEquals(10, [SELECT COUNT() FROM Account]);
  }
  
  @IsTest
  static void testUpdateAccounts() {
    // Create fresh data for this test
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 10; i++) {
      accounts.add(new Account(Name = 'Account ' + i));
    }
    insert accounts;
    
    for (Account acc : accounts) {
      acc.Revenue__c = 50000;
    }
    update accounts;
    
    System.assertEquals(10, [SELECT COUNT() FROM Account WHERE Revenue__c = 50000]);
  }
}
```

**Pros**: Tests are fully isolated, modifications don't affect other tests.

**Cons**: Data created for each test, slower.

**Best Practice**: Use @TestSetup for simple shared data, test method data when tests modify data.

---

## System.runAs for Permission Testing

### Test User Setup

```apex
@IsTest
private class PermissionTest {
  @TestSetup
  static void setupUsers() {
    // Create admin user
    User adminUser = new User(
      FirstName = 'Admin',
      LastName = 'User',
      Email = 'admin@example.com',
      Username = 'admin@example.com.' + System.now().millisecond(),
      ProfileId = [SELECT Id FROM Profile WHERE Name = 'System Administrator'].Id,
      Alias = 'admin',
      TimeZoneSidKey = 'America/Los_Angeles',
      LocaleSidKey = 'en_US',
      EmailEncodingKey = 'UTF-8',
      LanguageLocaleKey = 'en_US'
    );
    insert adminUser;
    
    // Create standard user
    User standardUser = new User(
      FirstName = 'Standard',
      LastName = 'User',
      Email = 'standard@example.com',
      Username = 'standard@example.com.' + System.now().millisecond(),
      ProfileId = [SELECT Id FROM Profile WHERE Name = 'Standard User'].Id,
      Alias = 'std',
      TimeZoneSidKey = 'America/Los_Angeles',
      LocaleSidKey = 'en_US',
      EmailEncodingKey = 'UTF-8',
      LanguageLocaleKey = 'en_US'
    );
    insert standardUser;
  }
  
  @IsTest
  static void testWithAdminUser() {
    User adminUser = [SELECT Id FROM User WHERE Email = 'admin@example.com' LIMIT 1];
    Account acc = new Account(Name = 'Test');
    
    System.runAs(adminUser) {
      insert acc;
      System.assertEquals('Test', [SELECT Name FROM Account WHERE Id = :acc.Id].Name);
    }
  }
  
  @IsTest
  static void testWithStandardUser() {
    User standardUser = [SELECT Id FROM User WHERE Email = 'standard@example.com' LIMIT 1];
    Account acc = [SELECT Id FROM Account LIMIT 1];
    
    System.runAs(standardUser) {
      // Test behavior as limited user
      try {
        acc.Name = 'Updated';
        update acc;
        System.assert(true, 'User has update permission');
      } catch (DmlException ex) {
        System.assert(false, 'User should have update permission');
      }
    }
  }
}
```

### Assign Permission Set to User

```apex
@IsTest
static void testWithFeaturePermission() {
  User testUser = [SELECT Id FROM User WHERE Email = 'standard@example.com'];
  
  // Assign permission set
  PermissionSetAssignment psa = new PermissionSetAssignment(
    PermissionSetId = [SELECT Id FROM PermissionSet WHERE Name = 'Feature_Access'].Id,
    AssigneeId = testUser.Id
  );
  insert psa;
  
  System.runAs(testUser) {
    // User now has feature access
    if (FeatureManagement.checkPermission('Feature_Name')) {
      System.assert(true, 'User has permission');
    }
  }
}
```

---

## Bulk Testing (200+ Records)

### Pattern 1: Simple Bulk

```apex
@IsTest
static void testBulkInsert() {
  List<Account> accounts = new List<Account>();
  for (Integer i = 0; i < 200; i++) {
    accounts.add(new Account(Name = 'Account ' + i));
  }
  
  Test.startTest();
  insert accounts;
  Test.stopTest();
  
  System.assertEquals(200, [SELECT COUNT() FROM Account]);
  System.assert(Limits.getQueries() < 100, 'Too many queries');
}
```

### Pattern 2: Bulk with Trigger Testing

```apex
@IsTest
static void testTriggerBulk() {
  List<Account> accounts = new List<Account>();
  for (Integer i = 0; i < 200; i++) {
    accounts.add(new Account(Name = 'Account ' + i, OwnerId = UserInfo.getUserId()));
  }
  
  Test.startTest();
  insert accounts;  // Trigger fires 200 times in parallel
  Test.stopTest();
  
  // Verify trigger created related records
  Integer taskCount = [SELECT COUNT() FROM Task];
  System.assertEquals(200, taskCount, 'Trigger should create 1 task per account');
}
```

---

## Edge Case Testing

### Null Values

```apex
@IsTest
static void testNullHandling() {
  Account acc = new Account(Name = null);  // Required field null
  
  try {
    insert acc;
    System.assert(false, 'Should fail with null');
  } catch (DmlException ex) {
    System.assert(ex.getMessage().contains('Required'));
  }
}
```

### Empty Collections

```apex
@IsTest
static void testEmptyList() {
  List<Account> emptyList = new List<Account>();
  
  // Service should handle gracefully
  List<Account> result = AccountService.processAccounts(emptyList);
  
  System.assertEquals(0, result.size());
}
```

### Duplicate Records

```apex
@IsTest
static void testDuplicate() {
  Account acc = new Account(Name = 'Acme');
  insert acc;
  
  // Try to insert duplicate
  Account duplicate = new Account(Name = 'Acme');
  insert duplicate;  // Should succeed (no unique constraint)
  
  List<Account> results = [SELECT Name FROM Account WHERE Name = 'Acme'];
  System.assertEquals(2, results.size());
}
```

### Boundary Values

```apex
@IsTest
static void testBoundaryValues() {
  // Max string length
  String longName = 'A'.repeat(255);
  Account acc = new Account(Name = longName);
  insert acc;
  
  System.assertEquals(255, [SELECT Name FROM Account WHERE Id = :acc.Id].Name.length());
  
  // Over limit (should fail)
  String tooLong = 'A'.repeat(256);
  Account acc2 = new Account(Name = tooLong);
  
  try {
    insert acc2;
    System.assert(false, 'Should fail');
  } catch (DmlException ex) {
    System.assert(true, 'Expected failure');
  }
}
```

---

## Assertion Patterns

### Basic Assertions

```apex
System.assertEquals(expected, actual);
System.assert(condition);
System.assertNotEquals(expected, actual);
```

### Collection Assertions

```apex
@IsTest
static void testCollections() {
  List<Account> accounts = [SELECT Id FROM Account];
  
  System.assert(!accounts.isEmpty(), 'Should have accounts');
  System.assertEquals(10, accounts.size(), 'Should have 10 accounts');
  System.assert(accounts.contains(expectedAcc), 'Should contain expected account');
}
```

### Exception Assertions

```apex
@IsTest
static void testException() {
  try {
    AccountService.throwError();
    System.assert(false, 'Should throw exception');
  } catch (AccountServiceException ex) {
    System.assertEquals('Expected error', ex.getMessage());
  }
}
```

---

## Checklist: Test Data Ready

- ✅ TestDataFactory used for reusable data
- ✅ @TestSetup for simple shared data
- ✅ Test methods isolated (own data where needed)
- ✅ System.runAs used for permission testing
- ✅ Bulk testing with 200+ records
- ✅ Edge cases tested (null, empty, boundary)
- ✅ Error paths tested (catch blocks)
- ✅ No hardcoded IDs in test data
- ✅ Governor limits checked (Limits.getQueries)
- ✅ Test assertions clear and specific

