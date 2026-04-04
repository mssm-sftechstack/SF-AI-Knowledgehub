# Apex Test Class Patterns

**90% coverage is the floor, not the target. A test with no assertions is worse than no test.**

---

## The Test Factory Pattern

One shared factory. Every test class uses it. Never duplicate data setup.

```apex
@IsTest
public class TestDataFactory {

    public static Account createAccount(Boolean doInsert) {
        Account acc = new Account(
            Name = 'Test Account',
            Industry = 'Technology',
            AnnualRevenue = 5000000
        );
        if (doInsert) insert acc;
        return acc;
    }

    public static List<Account> createAccounts(Integer count, Boolean doInsert) {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < count; i++) {
            accounts.add(new Account(
                Name = 'Test Account ' + i,
                Industry = 'Technology'
            ));
        }
        if (doInsert) insert accounts;
        return accounts;
    }

    public static User createStandardUser(Boolean doInsert) {
        Profile p = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
        User u = new User(
            FirstName = 'Test',
            LastName = 'User',
            Email = 'testuser' + Datetime.now().getTime() + '@example.com',
            Username = 'testuser' + Datetime.now().getTime() + '@example.com',
            Alias = 'tuser',
            TimeZoneSidKey = 'America/New_York',
            LocaleSidKey = 'en_US',
            EmailEncodingKey = 'UTF-8',
            LanguageLocaleKey = 'en_US',
            ProfileId = p.Id
        );
        if (doInsert) insert u;
        return u;
    }

    public static User createUserWithPermissionSet(String permSetName) {
        User u = createStandardUser(true);
        PermissionSet ps = [SELECT Id FROM PermissionSet WHERE Name = :permSetName LIMIT 1];
        insert new PermissionSetAssignment(AssigneeId = u.Id, PermissionSetId = ps.Id);
        return u;
    }
}
```

---

## Bulk Test — Always 200 Records

```apex
@IsTest
static void testSetRatingByRevenue_bulk() {
    List<Account> accounts = TestDataFactory.createAccounts(200, true);

    Test.startTest();
    AccountService service = new AccountService();
    service.setRatingByRevenue(accounts);
    update accounts;
    Test.stopTest();

    List<Account> result = [SELECT Id, Rating FROM Account WHERE Id IN :accounts];
    for (Account acc : result) {
        System.assertNotEquals(null, acc.Rating, 'Rating should be set for all accounts');
    }
}
```

Don't test with 1 record and call it done. Trigger frameworks, governor limits, and bulkification bugs only show up at scale.

---

## System.runAs() — When You Must Use It

Use it when your service enforces FLS or CRUD and the test user needs a permission set to see the fields.

```apex
@IsTest
static void testGetOpenCases_withFLS() {
    User testUser = TestDataFactory.createUserWithPermissionSet('Case_Manager');

    System.runAs(testUser) {
        Test.startTest();
        Map<Id, List<Case>> result = CaseService.getOpenCasesByAccount(accountIds);
        Test.stopTest();

        System.assertNotEquals(0, result.size(), 'Expected cases returned');
    }
}
```

If you call a service that uses `WITH USER_MODE` or `Security.stripInaccessible()` as System Admin in a test, the FLS check passes trivially and the test proves nothing. Run it as a Standard User with only the intended permission set.

---

## Mixed DML Trap

Inserting a User (setup object) and a non-setup object like Account in the same test method throws `MIXED_DML_OPERATION`.

**Wrong:**
```apex
@IsTest
static void testMixedDml_wrong() {
    User u = new User(/* ... */);
    insert u; // setup object

    Account acc = new Account(Name = 'Test');
    insert acc; // non-setup object: MIXED_DML_OPERATION error
}
```

Fix: wrap the setup object DML in `System.runAs()`.

```apex
// RIGHT
@IsTest
static void testMixedDml_fixed() {
    User u;
    System.runAs(new User(Id = UserInfo.getUserId())) {
        u = new User(/* ... */);
        insert u;
    }

    Account acc = new Account(Name = 'Test');
    insert acc; // now safe
}
```

Or just use `TestDataFactory.createStandardUser(true)` — it already handles this.

---

## Callout Mocking

Never make real callouts in tests. They fail with "You have uncommitted work pending".

**Option 1: StaticResourceCalloutMock**

Upload a JSON file as a Static Resource, then reference it:

```apex
@IsTest
static void testCallout_staticMock() {
    StaticResourceCalloutMock mock = new StaticResourceCalloutMock();
    mock.setStaticResource('MockCaseApiResponse'); // Static Resource name
    mock.setStatusCode(200);
    mock.setHeader('Content-Type', 'application/json');
    Test.setMock(HttpCalloutMock.class, mock);

    Test.startTest();
    CaseApiService.fetchCases('ACC-001');
    Test.stopTest();
}
```

**Option 2: Inline HttpCalloutMock**

```apex
@IsTest
static void testCallout_inlineMock() {
    Test.setMock(HttpCalloutMock.class, new MockHttpResponse(200, '{"cases":[]}'));

    Test.startTest();
    CaseApiService.fetchCases('ACC-001');
    Test.stopTest();
}

@IsTest
private class MockHttpResponse implements HttpCalloutMock {
    private Integer statusCode;
    private String body;

    public MockHttpResponse(Integer statusCode, String body) {
        this.statusCode = statusCode;
        this.body = body;
    }

    public HttpResponse respond(HttpRequest req) {
        HttpResponse res = new HttpResponse();
        res.setStatusCode(statusCode);
        res.setBody(body);
        return res;
    }
}
```

Use `StaticResourceCalloutMock` for complex JSON payloads. Use inline for simple or error scenarios.

---

## What to Assert

Don't assert "no exception thrown". Assert the actual outcome.

**Wrong:**
```apex
@IsTest
static void testCreateCase_wrong() {
    Test.startTest();
    CaseService.createCase(accountId, 'Test subject');
    Test.stopTest();
    // no asserts
}
```

**Right:**
```apex
@IsTest
static void testCreateCase_right() {
    Id accountId = TestDataFactory.createAccount(true).Id;

    Test.startTest();
    Case c = CaseService.createCase(accountId, 'Test subject');
    Test.stopTest();

    System.assertNotEquals(null, c.Id, 'Case should be inserted');
    Case result = [SELECT Id, Subject, Status, AccountId FROM Case WHERE Id = :c.Id];
    System.assertEquals('Test subject', result.Subject, 'Subject mismatch');
    System.assertEquals('New', result.Status, 'Status should default to New');
    System.assertEquals(accountId, result.AccountId, 'AccountId mismatch');
}
```

---

## @TestSetup vs Inline Setup

| Approach | Use When |
|---|---|
| `@TestSetup` | Multiple test methods need the same data set. Runs once, data committed before each test method. |
| Inline setup | One-off data specific to a single test. Or when you need different data per test. |

```apex
@IsTest
private class AccountServiceTest {

    @TestSetup
    static void setup() {
        // runs once, shared by all test methods below
        TestDataFactory.createAccounts(200, true);
    }

    @IsTest
    static void testGetByIndustry() {
        // accounts from @TestSetup are already in the database
        List<Account> accounts = new AccountSelector().getByIndustry('Technology');
        System.assertEquals(200, accounts.size(), 'Should return all 200 accounts');
    }
}
```

One gotcha with `@TestSetup`: you can't insert Users in `@TestSetup` if you also insert non-setup objects in the same method. Split them with `System.runAs()`.

---

## Schedulable Testing

Never test a callout-enabled schedulable through `System.schedule()` + `Test.stopTest()`. That triggers `execute()` inside `stopTest()`, which throws "Callout from scheduled Apex not supported".

**Wrong:**
```apex
@IsTest
static void testSchedulable_wrong() {
    Test.startTest();
    System.schedule('Test Job', '0 0 * * * ?', new CaseSyncSchedulable());
    Test.stopTest(); // execute() fires here, callout fails
}
```

**Right:**
```apex
@IsTest
static void testSchedulable_right() {
    Test.setMock(HttpCalloutMock.class, new MockHttpResponse(200, '{}'));

    Test.startTest();
    new CaseSyncSchedulable().execute(null);
    Test.stopTest();

    // assert side effects
}
```

---

## Complete Well-Structured Test Class

```apex
@IsTest(SeeAllData=false)
private class AccountServiceTest {

    @TestSetup
    static void setup() {
        TestDataFactory.createAccounts(200, true);
    }

    @IsTest
    static void testSetRatingByRevenue_hot() {
        Account acc = new Account(Name = 'Hot Co', AnnualRevenue = 15000000);
        insert acc;

        Test.startTest();
        new AccountService().setRatingByRevenue(new List<Account>{ acc });
        Test.stopTest();

        System.assertEquals('Hot', acc.Rating, 'Expected Hot rating for revenue >= 10M');
    }

    @IsTest
    static void testSetRatingByRevenue_nullRevenue() {
        Account acc = new Account(Name = 'No Revenue Co', AnnualRevenue = null);

        Test.startTest();
        new AccountService().setRatingByRevenue(new List<Account>{ acc });
        Test.stopTest();

        System.assertEquals(null, acc.Rating, 'Rating should stay null when revenue is null');
    }

    @IsTest
    static void testSetRatingByRevenue_bulk() {
        List<Account> accounts = [SELECT Id, AnnualRevenue, Rating FROM Account LIMIT 200];
        for (Account acc : accounts) {
            acc.AnnualRevenue = 12000000;
        }

        Test.startTest();
        new AccountService().setRatingByRevenue(accounts);
        update accounts;
        Test.stopTest();

        List<Account> updated = [SELECT Id, Rating FROM Account LIMIT 200];
        for (Account acc : updated) {
            System.assertEquals('Hot', acc.Rating, 'All bulk accounts should be Hot');
        }
    }
}
```
