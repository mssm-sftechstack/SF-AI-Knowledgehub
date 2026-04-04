# Callout Patterns

**Never call an external API from trigger context. Everything else about callouts comes after that rule.**

---

## The Queueable Pattern for Trigger-Initiated Callouts

Triggers can't make callouts. You need an async boundary. Queueable with `Database.AllowsCallouts` is the right choice when you need both DML and callouts in the same job.

```apex
// From the trigger/service, enqueue the job
System.enqueueJob(new CaseApiQueueable(caseIds));

// CaseApiQueueable.cls
public class CaseApiQueueable implements Queueable, Database.AllowsCallouts {

    private Set<Id> caseIds;

    public CaseApiQueueable(Set<Id> caseIds) {
        this.caseIds = caseIds;
    }

    public void execute(QueueableContext ctx) {
        List<Case> cases = [
            SELECT Id, Subject, Status, AccountId
            FROM Case
            WHERE Id IN :caseIds
            WITH SECURITY_ENFORCED
        ];

        for (Case c : cases) {
            HttpResponse res = callExternalApi(c);
            if (res == null || res.getStatusCode() >= 500) {
                // log error, optionally re-enqueue with retry logic
                logError(c.Id, 'API call failed or timed out');
                continue;
            }
            if (res.getStatusCode() == 200) {
                processResponse(c, res.getBody());
            }
        }
    }

    private HttpResponse callExternalApi(Case c) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:Case_External_API/cases');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(JSON.serialize(new Map<String, Object>{
            'caseId' => c.Id,
            'subject' => c.Subject
        }));
        req.setTimeout(10000); // 10 seconds, explicit

        try {
            return new Http().send(req);
        } catch (System.CalloutException e) {
            logError(c.Id, e.getMessage());
            return null;
        }
    }

    private void logError(Id caseId, String message) {
        insert new Error_Log__c(
            Error_Message__c = message,
            Context__c = 'CaseApiQueueable - case ' + caseId,
            Occurred_At__c = Datetime.now()
        );
    }

    private void processResponse(Case c, String body) {
        if (String.isBlank(body)) return; // null-check before deserialise
        Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(body);
        // update case fields from response
    }
}
```

---

## HTTP Status Handling

| Status | Meaning | Action |
|---|---|---|
| 200-299 | Success | Process response body |
| 400 | Bad request | Log the error and fail. Don't retry. Fix the payload. |
| 401 | Unauthorized | Re-authenticate once, retry once. If still 401, alert and stop. |
| 403 | Forbidden | Log and fail. Retrying won't help. Permissions issue. |
| 404 | Not found | Log and fail. The resource doesn't exist. |
| 429 | Rate limited | Respect `Retry-After` header. Enqueue a new job with delay. |
| 500-599 | Server error | Retry with exponential backoff. Max 3 attempts. |

---

## Timeout

Always set it explicitly. The default is 10 seconds. Max is 120 seconds. Set it to match the API's documented SLA.

```apex
req.setTimeout(30000); // 30 seconds for a slow partner API
```

If you don't set a timeout and the API hangs, your Queueable burns its entire CPU limit waiting.

---

## Null-Check the Response Body

A timeout or network error returns a null body. `JSON.deserializeUntyped(null)` throws a `JSONException`.

**Wrong:**
```apex
Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
```

**Right:**
```apex
String body = res.getBody();
if (String.isBlank(body)) {
    logError(recordId, 'Empty response body from API');
    return;
}
Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(body);
```

---

## Named Credentials

Every callout endpoint goes through a Named Credential. No hardcoded URLs. No credentials in code.

**Wrong:**
```apex
req.setEndpoint('https://api.example.com/cases');
req.setHeader('Authorization', 'Bearer mytoken123');
```

**Right:**
```apex
req.setEndpoint('callout:Case_External_API/cases');
// auth header injected automatically from the Named Credential
```

The `callout:` prefix tells Salesforce to look up the Named Credential named `Case_External_API` and prepend its URL. Auth headers are injected from the credential, not your code.

---

## Callout Mocks for Tests

**StaticResourceCalloutMock** for complex payloads stored as a Static Resource:

```apex
@IsTest
static void testQueueable_success() {
    StaticResourceCalloutMock mock = new StaticResourceCalloutMock();
    mock.setStaticResource('MockCaseApiResponse'); // Static Resource name
    mock.setStatusCode(200);
    mock.setHeader('Content-Type', 'application/json');
    Test.setMock(HttpCalloutMock.class, mock);

    List<Id> caseIds = new List<Id>{ TestDataFactory.createCase(true).Id };

    Test.startTest();
    System.enqueueJob(new CaseApiQueueable(new Set<Id>(caseIds)));
    Test.stopTest();

    // assert side effects
}
```

**Inline HttpCalloutMock** for simple or error scenarios:

```apex
@IsTest
private class MockHttpResponse implements HttpCalloutMock {
    private Integer status;
    private String body;

    public MockHttpResponse(Integer status, String body) {
        this.status = status;
        this.body = body;
    }

    public HttpResponse respond(HttpRequest req) {
        HttpResponse res = new HttpResponse();
        res.setStatusCode(status);
        res.setBody(body);
        return res;
    }
}

@IsTest
static void testQueueable_apiError() {
    Test.setMock(HttpCalloutMock.class, new MockHttpResponse(500, 'Internal Server Error'));

    Test.startTest();
    new CaseApiQueueable(new Set<Id>{ someCase.Id }).execute(null);
    Test.stopTest();

    List<Error_Log__c> logs = [SELECT Id FROM Error_Log__c];
    System.assertEquals(1, logs.size(), 'Error should be logged on 500 response');
}
```

Test the 200 path, the 400/500 error paths, and the null body path. Each is a separate test method.
