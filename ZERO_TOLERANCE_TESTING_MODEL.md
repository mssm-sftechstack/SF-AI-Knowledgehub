# Zero-Tolerance Testing Model

**CRITICAL DIRECTIVE FOR AI:** Your goal is not code coverage. Your goal is state verification. Never generate a test class that simply calls a method and exits. You must prove the code works under bulk conditions and failure scenarios.

## 1. The AAA Pattern is Mandatory
Every test method must clearly follow the Arrange, Act, Assert pattern.

1. **Arrange:** Create isolated test data. NEVER use `(SeeAllData=true)`.
2. **Act:** Execute the logic wrapped inside `Test.startTest()` and `Test.stopTest()`.
3. **Assert:** Query the database post-execution and use `Assert.areEqual()` to verify the exact state change.

## 2. Bulkification Testing
AI models default to testing a single record. This is a fatal flaw in Salesforce.
* Every trigger or bulk-capable method MUST be tested with a collection of at least 200 records to ensure governor limits (like SOQL 101) are not breached.

## 3. The HTTP Callout Mock Mandate
AI models frequently forget that Salesforce strictly blocks real outbound HTTP callouts during test execution. 
* **The Rule:** If testing a class that performs an integration, you MUST generate and set an `HttpCalloutMock` class before calling `Test.startTest()`.

```apex
// ✅ MANDATORY CALLOUT MOCK PATTERN
@isTest
static void testExternalCallout() {
    // Arrange: Set the mock
    Test.setMock(HttpCalloutMock.class, new StandardResponseMock(200, '{"status":"success"}'));
    
    // Act
    Test.startTest();
    MyIntegrationService.syncData();
    Test.stopTest();
    
    // Assert
    // Verify internal state changes based on the mock response
}
```

## 4. Negative Testing (The "Happy Path" Fallacy)
You must generate tests that intentionally fail.
* Pass null lists.
* Pass records missing required fields to test custom exceptions or `try/catch` block handling.
* Assert that the correct error messages are added to the page or logged.