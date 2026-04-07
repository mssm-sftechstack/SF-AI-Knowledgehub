# Testing Strategy

Comprehensive guide to testing Apex, LWC, and Flows. Covers unit vs. integration testing, bulk testing, edge cases, and achieving 90%+ coverage.

---

## Why 90%+ Coverage Matters

Code coverage percentage is misleading. A class with 95% coverage can still have critical bugs if the untested 5% is error handling.

**Real Coverage = Code Lines Tested + Edge Cases + Error Paths**

```apex
public class AccountService {
  public static void updateAccounts(List<Account> accounts) {
    update accounts;  // 1 line, 100% coverage if we just insert and update
  }
}
```

If we only test the happy path, we cover 100% but don't test:
- Empty lists
- Null values
- Duplicate records
- Records without required fields
- DML errors
- Permission issues

**Minimum 90% coverage means**: All code paths tested (if/else, catch blocks, loops) AND edge cases.

---

## Unit vs. Integration Testing

### Unit Testing

Tests a single method in isolation. Dependencies are mocked.

```apex
@IsTest
private class AccountServiceTest {
  @IsTest
  static void testCalculateAnnualRevenue() {
    // Arrange: Create test data
    Account acc = new Account(Name = 'Acme', Revenue__c = 100000);
    
    // Act: Call method
    Decimal revenue = AccountService.calculateAnnualRevenue(acc);
    
    // Assert: Verify result
    System.assertEquals(100000, revenue, 'Revenue should match');
  }
}
```

**Pros**: Fast, isolated, easy to debug.

**Cons**: May miss integration issues (database, triggers, flows).

### Integration Testing

Tests multiple components working together. Uses real database.

```apex
@IsTest
private class AccountIntegrationTest {
  @IsTest
  static void testCreateAccountTriggersFlow() {
    // Create account (triggers before-insert)
    Account acc = new Account(Name = 'Acme');
    insert acc;  // Triggers after-insert flow
    
    // Verify related records created by trigger/flow
    List<Task> tasks = [SELECT Id FROM Task WHERE WhoId = :acc.OwnerId];
    System.assert(!tasks.isEmpty(), 'Flow should create task');
  }
}
```

**Pros**: Tests real behavior, catches integration bugs.

**Cons**: Slower, harder to debug, depends on org setup.

---

## Test Data Patterns

### Pattern 1: TestSetup Method

```apex
@IsTest
private class AccountServiceTest {
  @TestSetup
  static void setupTestData() {
    // Data created once per class, shared across all test methods
    Account acc = new Account(Name = 'Shared Account');
    insert acc;
  }
  
  @IsTest
  static void testUpdateAccount() {
    Account acc = [SELECT Id FROM Account LIMIT 1];
    acc.Name = 'Updated';
    update acc;
    
    System.assertEquals('Updated', [SELECT Name FROM Account WHERE Id = :acc.Id].Name);
  }
}
```

**Use TestSetup when**: Multiple test methods use the same data.

### Pattern 2: Test Data Factory

```apex
@IsTest
public class TestDataFactory {
  public static Account createAccount(String name) {
    return new Account(Name = name);
  }
  
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
static void testBulkUpdate() {
  List<Account> accounts = TestDataFactory.createAccounts(200);
  insert accounts;
  
  // Update all
  for (Account acc : accounts) {
    acc.Revenue__c = 50000;
  }
  update accounts;
  
  System.assertEquals(200, [SELECT COUNT() FROM Account WHERE Revenue__c = 50000]);
}
```

**Use Factory when**: Creating multiple variations of test data.

### Pattern 3: System.runAs for Permission Testing

```apex
@IsTest
private class AccountServiceSecurityTest {
  @TestSetup
  static void setupUsers() {
    // Create test user with limited permissions
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
  }
  
  @IsTest
  static void testUpdateWithoutPermission() {
    User testUser = [SELECT Id FROM User WHERE Email = 'testuser@example.com' LIMIT 1];
    Account acc = [SELECT Id FROM Account LIMIT 1];
    
    // Run as limited user
    System.runAs(testUser) {
      acc.Name = 'Updated';
      
      try {
        update acc;
        System.assert(false, 'Should fail due to permissions');
      } catch (DmlException ex) {
        System.assert(ex.getMessage().contains('INSUFFICIENT_ACCESS'));
      }
    }
  }
}
```

---

## Bulk Testing (200+ Records)

Bulk tests ensure code works at scale.

```apex
@IsTest
private class AccountServiceBulkTest {
  @IsTest
  static void testUpdateBulkAccounts() {
    // Create 200 accounts
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
      accounts.add(new Account(Name = 'Account ' + i));
    }
    insert accounts;
    
    // Update all (triggers should handle bulk)
    for (Account acc : accounts) {
      acc.Revenue__c = 50000;
    }
    update accounts;
    
    // Verify all updated
    Integer count = [SELECT COUNT() FROM Account WHERE Revenue__c = 50000];
    System.assertEquals(200, count, 'All 200 accounts should update');
    
    // Verify no SOQL or DML limits hit
    System.assert(Limits.getQueries() < 100, 'Too many SOQL queries');
    System.assert(Limits.getDmlRows() <= 200, 'Too much DML');
  }
}
```

**Why bulk testing?**
- Developers test with 50 records. Production has 50,000.
- Code that works for 50 may fail at 200+ due to SOQL/DML limits.
- Bulk tests catch these issues before production.

---

## Testing Async Code

### Testing Queueable Jobs

```apex
@IsTest
private class UpdateExternalSystemQueueableTest {
  @IsTest
  static void testQueueableExecution() {
    Account acc = new Account(Name = 'Test Acme');
    insert acc;
    
    Test.startTest();
    
    // Enqueue job
    System.enqueueJob(new UpdateExternalSystemQueueable(new List<Account>{acc}));
    
    Test.stopTest();  // Force async jobs to execute in test
    
    // Verify result
    System.assertEquals('Updated', [SELECT Name FROM Account WHERE Id = :acc.Id].Name);
  }
}
```

### Testing Scheduled Jobs

```apex
@IsTest
private class RefreshCoverageSchedulableTest {
  @IsTest
  static void testSchedulableExecution() {
    Test.startTest();
    
    // Execute the schedulable directly (avoid Test.schedule)
    RefreshCoverageSchedulable schedulable = new RefreshCoverageSchedulable();
    SchedulableContext mockContext = null;
    schedulable.execute(mockContext);  // Call execute() directly
    
    Test.stopTest();
    
    // Verify expected result
    Integer logCount = [SELECT COUNT() FROM CodeCoverageLog__c];
    System.assert(logCount > 0, 'Should have logged coverage');
  }
}
```

**Important**: Do NOT use Test.schedule() for callout-enabled schedulables. Call execute() directly instead.

### Testing Batch Jobs

```apex
@IsTest
private class UpdateAccountsBatchTest {
  @IsTest
  static void testBatchExecution() {
    // Create test data
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 250; i++) {
      accounts.add(new Account(Name = 'Account ' + i));
    }
    insert accounts;
    
    Test.startTest();
    
    // Execute batch with batch size
    Database.executeBatch(new UpdateAccountsBatch(), 200);
    
    Test.stopTest();
    
    // Verify all updated
    System.assertEquals(250, [SELECT COUNT() FROM Account WHERE Updated__c = true]);
  }
}
```

---

## Testing Flows

### Manual Testing Steps

1. Go to Flow Builder
2. Run the flow manually (Flow → Details → Run)
3. Provide inputs
4. Verify outputs and side effects

### Automated Testing (Apex)

Flows are hard to test in Apex. Instead, test the Apex triggered by flows.

```apex
@IsTest
private class AccountFlowIntegrationTest {
  @IsTest
  static void testAccountFlowLogic() {
    // Create account (triggers record-triggered flow)
    Account acc = new Account(Name = 'Test');
    insert acc;  // Flow fires after insert
    
    // Verify flow updated related records
    List<Task> tasks = [SELECT Id FROM Task WHERE WhoId = :acc.OwnerId];
    System.assert(!tasks.isEmpty(), 'Flow should create task');
  }
}
```

---

## Testing LWC with Jest

Jest tests LWC in isolation (no Apex, no Salesforce org).

### Setup

```bash
npm install --save-dev @salesforce/sfdx-lwc-jest jest
```

### Test Example

```javascript
import { LightningElement } from 'lwc';
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';

describe('MyComponent', () => {
  let element;
  
  beforeEach(() => {
    element = createElement('c-my-component', { is: MyComponent });
    document.body.appendChild(element);
  });
  
  afterEach(() => {
    while (document.body.firstChild) {
      document.body.removeChild(document.body.firstChild);
    }
  });
  
  it('renders button', () => {
    const button = element.shadowRoot.querySelector('button');
    expect(button).toBeTruthy();
  });
  
  it('increments counter on click', () => {
    const button = element.shadowRoot.querySelector('button');
    button.click();
    
    expect(element.counter).toBe(1);
  });
});
```

---

## Edge Case Testing

### Null Values

```apex
@IsTest
static void testWithNullValues() {
  Account acc = new Account(Name = null);  // Required field null
  
  try {
    insert acc;
    System.assert(false, 'Should fail with null Name');
  } catch (DmlException ex) {
    System.assert(ex.getMessage().contains('Required'));
  }
}
```

### Empty Collections

```apex
@IsTest
static void testWithEmptyList() {
  List<Account> accounts = new List<Account>();
  
  // Service should handle empty list gracefully
  List<Account> result = AccountService.updateAccounts(accounts);
  
  System.assertEquals(0, result.size());
}
```

### Duplicate Records

```apex
@IsTest
static void testDuplicateHandling() {
  Account acc = new Account(Name = 'Acme', External_Id__c = 'EXT123');
  insert acc;
  
  // Try to insert duplicate
  Account duplicate = new Account(Name = 'Acme', External_Id__c = 'EXT123');
  
  try {
    insert duplicate;
    System.assert(false, 'Should fail on duplicate');
  } catch (DmlException ex) {
    System.assert(ex.getMessage().contains('duplicate'));
  }
}
```

### Large Numbers

```apex
@IsTest
static void testLargeNumberCalculation() {
  Decimal largeNumber = 999999999999999999D;
  Decimal result = AccountService.calculateTotal(largeNumber);
  
  System.assertEquals(largeNumber, result);
}
```

---

## Governor Limit Testing

```apex
@IsTest
static void testQueryLimit() {
  // Create 100+ accounts to test query limits
  List<Account> accounts = new List<Account>();
  for (Integer i = 0; i < 100; i++) {
    accounts.add(new Account(Name = 'Account ' + i));
  }
  insert accounts;
  
  Test.startTest();
  
  // Method that queries accounts in loop
  AccountService.processAccounts(accounts);
  
  Test.stopTest();
  
  // Assert within limits
  Integer queries = Limits.getQueries();
  System.assert(queries < 100, 'Exceeded 100 SOQL query limit: ' + queries);
}
```

---

## Test Class Checklist

- ✅ Test method names describe what's being tested
- ✅ All public methods have at least one test
- ✅ Happy path tested
- ✅ Error paths tested (try-catch)
- ✅ Edge cases tested (null, empty, large)
- ✅ Bulk testing (200+ records)
- ✅ Permission testing (System.runAs with limited user)
- ✅ Governor limits checked
- ✅ 90%+ code coverage
- ✅ SeeAllData=false (tests are org-agnostic)
- ✅ No hardcoded IDs in tests
- ✅ Test data cleaned up (or @TestSetup used)

