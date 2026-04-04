# Salesforce Apex Test Skill

**Test class generation and review. Enforces 90%+ coverage, bulk testing at 200 records, proper mocking, Mixed DML handling, and runAs patterns.**

## Standards

- Minimum **90% coverage**. Target 95%+.
- Always use `@IsTest(SeeAllData=false)` on the class declaration.
- Use `@TestSetup` for shared test data -- reduces DML per test method.
- Every method covers: positive path, negative path, bulk (200 records), edge cases.
- Use `Test.startTest()` / `Test.stopTest()` in every test method -- resets governor limits.
- Mock ALL callouts with `HttpCalloutMock` -- never make real HTTP calls in tests.
- Never use `@IsTest(SeeAllData=true)`.
- Use `@TestVisible` (not `global`) to expose private helpers for testing.

**Reference:** [Apex Testing Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing.htm)

> **Order of Execution in Tests**
> Test execution runs the full 14-step order of execution sequence for every save. Before-save Flows fire at step 2, before-triggers at step 3, after-triggers at step 7, after-save Flows at step 11. When your test inserts 200 records, all validation rules, workflow rules, and Flows fire too. Design your test data to avoid triggering unrelated automations, or your tests will fail for reasons unrelated to your code.

---

## Test Class Template

```apex
@IsTest(SeeAllData=false)
private class AccountServiceTest {

    @TestSetup
    static void setup() {
        // insert shared data once -- all test methods share via SOQL
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Test Account ' + i));
        }
        insert accounts;
    }

    // positive path
    @IsTest
    static void processAccounts_happyPath() {
        List<Account> accounts = [SELECT Id FROM Account LIMIT 10];

        Test.startTest();
        AccountService.processAccounts(new Map<Id, Account>(accounts).keySet());
        Test.stopTest();

        List<Account> updated = [SELECT Status__c FROM Account WHERE Id IN :accounts];
        for (Account a : updated) {
            System.assertEquals('Processed', a.Status__c, 'status should be set');
        }
    }

    // negative path
    @IsTest
    static void processAccounts_emptyList_noOp() {
        Test.startTest();
        AccountService.processAccounts(new Set<Id>());
        Test.stopTest();
        System.assertEquals(0, Limits.getDmlStatements(), 'no DML for empty input');
    }

    // bulk -- always 200
    @IsTest
    static void processAccounts_bulk200() {
        Set<Id> ids = new Map<Id, Account>(
            [SELECT Id FROM Account LIMIT 200]
        ).keySet();

        Test.startTest();
        AccountService.processAccounts(ids);
        Test.stopTest();

        Integer processed = [SELECT COUNT() FROM Account WHERE Status__c = 'Processed'];
        System.assertEquals(200, processed, 'all 200 should be processed');
    }

    // edge case -- null input
    @IsTest
    static void processAccounts_nullInput_noOp() {
        Test.startTest();
        AccountService.processAccounts(null);
        Test.stopTest();
        System.assertEquals(0, Limits.getDmlStatements(), 'no DML for null input');
    }
}
```

---

## Mixed DML Exception

Mixed DML occurs when a test inserts setup objects (User, Profile, etc.) and regular objects in the same transaction. Wrap the regular DML in a `System.runAs()` block.

```apex
@IsTest
static void setup_newUser_noMixedDml() {
    Profile p = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
    User u = new User(
        LastName       = 'Test',
        Email          = 'test@example.com',
        Username       = 'test' + System.now().getTime() + '@example.com',
        Alias          = 'tstu',
        TimeZoneSidKey = 'America/Los_Angeles',
        LocaleSidKey   = 'en_US',
        EmailEncodingKey  = 'UTF-8',
        LanguageLocaleKey = 'en_US',
        ProfileId      = p.Id
    );
    insert u;

    System.runAs(u) {
        Test.startTest();
        insert new Account(Name = 'Test Account');
        Test.stopTest();

        Account acc = [SELECT OwnerId FROM Account LIMIT 1];
        System.assertEquals(u.Id, acc.OwnerId, 'new user should own the account');
    }
}
```

**Reference:** [Mixed DML Operations](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_dml_non_mix_sobjects.htm)

---

## @TestVisible -- Testing Private Helpers

```apex
public with sharing class WeatherService {

    @TestVisible
    private static String buildUrl(String city) {
        return 'https://api.example.com/weather?q=' + city;
    }
}

@IsTest
static void buildUrl_formatsCorrectly() {
    String url = WeatherService.buildUrl('London');
    System.assert(url.contains('London'), 'city name should be in URL');
}
```

Never use `global` just to make something testable.

**Reference:** [TestVisible Annotation](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_testvisible.htm)

---

## Callout Mock

```apex
// inner class mock -- private is enough
private class SuccessMock implements HttpCalloutMock {
    public HttpResponse respond(HttpRequest req) {
        HttpResponse res = new HttpResponse();
        res.setStatusCode(200);
        res.setHeader('Content-Type', 'application/json');
        res.setBody('{"id":"001","success":true}');
        return res;
    }
}

// multi-response mock for different endpoints
private class MultiMock implements HttpCalloutMock {
    private Map<String, HttpCalloutMock> mocks;
    MultiMock(Map<String, HttpCalloutMock> mocks) { this.mocks = mocks; }

    public HttpResponse respond(HttpRequest req) {
        String endpoint = req.getEndpoint();
        for (String key : mocks.keySet()) {
            if (endpoint.contains(key)) return mocks.get(key).respond(req);
        }
        throw new IllegalArgumentException('No mock for: ' + endpoint);
    }
}

Test.setMock(HttpCalloutMock.class, new SuccessMock());
```

**Reference:** [Testing HTTP Callouts](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_restful_http_testing_httpcalloutmock.htm)

---

## Test Data Factory

```apex
@IsTest
public class TestDataFactory {

    public static Account makeAccount(Boolean doInsert) {
        Account a = new Account(Name = 'Test Account', BillingCountry = 'US');
        if (doInsert) insert a;
        return a;
    }

    public static List<Opportunity> makeOpportunities(Id accountId, Integer count, Boolean doInsert) {
        List<Opportunity> opps = new List<Opportunity>();
        for (Integer i = 0; i < count; i++) {
            opps.add(new Opportunity(
                Name        = 'Opp ' + i,
                AccountId   = accountId,
                StageName   = 'Prospecting',
                CloseDate   = Date.today().addDays(30)
            ));
        }
        if (doInsert) insert opps;
        return opps;
    }
}
```

---

## Testing Async Apex

```apex
// Future methods
@IsTest
static void myFutureMethod_test() {
    Test.startTest();
    MyClass.myFutureMethod(recordId);
    Test.stopTest();
    // assert after stopTest -- future has completed by then
}

// Queueable
@IsTest
static void myQueueable_test() {
    Test.startTest();
    System.enqueueJob(new MyQueueable(ids));
    Test.stopTest();
}

// Batch Apex
@IsTest
static void myBatch_test() {
    insert testRecords;
    Test.startTest();
    Database.executeBatch(new MyBatch(), 200);
    Test.stopTest();
    // assert post-batch state
}

// Schedulable -- test by calling execute() directly, not System.schedule()
// Never test callout-enabled schedulables through System.schedule() + Test.stopTest()
// -- it triggers execute() and throws "Callout from scheduled Apex not supported"
@IsTest
static void myScheduled_test() {
    Test.startTest();
    new MyScheduled().execute(null);
    Test.stopTest();
    // assert expected state
}
```

**Reference:** [Testing Async Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_async_methods.htm)

---

## Testing with runAs (FLS / Permission Tests)

```apex
@IsTest
static void save_noPermission_throws() {
    Profile p = [SELECT Id FROM Profile WHERE Name = 'Minimum Access - Salesforce' LIMIT 1];
    User u = new User(
        LastName       = 'Test',
        Email          = 'test@example.com',
        Username       = 'test' + System.now().getTime() + '@example.com',
        Alias          = 'tstu',
        TimeZoneSidKey = 'America/Los_Angeles',
        LocaleSidKey   = 'en_US',
        EmailEncodingKey = 'UTF-8',
        LanguageLocaleKey = 'en_US',
        ProfileId      = p.Id
    );
    insert u;

    System.runAs(u) {
        Test.startTest();
        try {
            MyService.saveRecord(data);
            System.assert(false, 'should have thrown due to no permission');
        } catch (AuraHandledException e) {
            System.assert(e.getMessage().containsIgnoreCase('permission'), 'wrong error: ' + e.getMessage());
        }
        Test.stopTest();
    }
}
```

**Reference:** [System.runAs](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_tools_runas.htm)

---

## Coverage Checklist

- [ ] 90%+ line coverage on the class under test
- [ ] `@IsTest(SeeAllData=false)` on class declaration
- [ ] All public/global methods have at least one positive test
- [ ] All error paths have negative tests
- [ ] Bulk test with exactly 200 records
- [ ] Callout methods use `HttpCalloutMock`
- [ ] Async methods wrapped in `Test.startTest()` / `Test.stopTest()`
- [ ] All DML operations verified by querying back
- [ ] `@TestVisible` used to test private helpers (not `global`)
- [ ] Mixed DML avoided -- setup object inserts wrapped in `System.runAs()`

---

## Running Tests

```bash
# single class with coverage
sf apex run test --class-names AccountServiceTest --target-org <your-org-alias> --code-coverage --wait 10

# multiple classes
sf apex run test --class-names "AccountServiceTest,WeatherServiceTest" --target-org <your-org-alias> --wait 20

# all local tests -- run before every deployment
sf apex run test --test-level RunLocalTests --code-coverage --target-org <your-org-alias> --wait 60

# check results after async run
sf apex get test --test-run-id <jobId> --code-coverage --target-org <your-org-alias>
```

---

## Sources

- [Apex Testing Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing.htm)
- [Apex Test Best Practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_best_practices.htm)
- [Testing HTTP Callouts](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_restful_http_testing_httpcalloutmock.htm)
- [Testing Async Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_async_methods.htm)
- [System.runAs](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_tools_runas.htm)
- [Mixed DML Operations](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_dml_non_mix_sobjects.htm)
- [Well-Architected: Resilient](https://architect.salesforce.com/well-architected/adaptable/resilient)
