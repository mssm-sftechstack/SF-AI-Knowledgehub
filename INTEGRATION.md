# Integration Patterns

Covering callouts, Named Credentials, retry logic, and error handling for external system integrations.

---

## Rule: Never Callout from Trigger Context

This is a hard rule. Triggers cannot make HTTP callouts.

```apex
// ❌ Wrong
trigger AccountTrigger on Account (after insert) {
  callExternalSystem(Trigger.new);  // Fails at runtime
}

// ✅ Right
trigger AccountTrigger on Account (after insert) {
  System.enqueueJob(new UpdateExternalSystemQueueable(Trigger.new));
}
```

**Why**: Triggers are synchronous. Callouts are asynchronous. Salesforce blocks callouts in trigger context to prevent deadlocks and timeouts.

**Solution**: Use Queueable with `Database.AllowsCallouts` interface.

---

## Named Credentials: Setup & Best Practices

Named Credentials store authentication details securely. Never hardcode credentials.

### Setup Steps

1. Go to Setup > Integrations > Named Credentials
2. Click New
3. **Label**: `ExternalSystem`
4. **Name**: `ExternalSystem` (must match in Apex code)
5. **URL**: `https://api.example.com`
6. **Auth Type**:
   - **OAuth 2.0**: For APIs supporting OAuth (Google, Salesforce)
   - **Basic**: For username/password
   - **Custom Header**: For API keys
7. **Auth Provider**: (if OAuth) Select or create
8. **Scope**: (if OAuth) As per API requirements (e.g., `https://www.googleapis.com/auth/contacts.readonly`)

### In Apex Code

```apex
public class UpdateExternalSystemQueueable implements Queueable, Database.AllowsCallouts {
  private List<Account> accounts;
  
  public void execute(QueueableContext context) {
    try {
      for (Account acc : accounts) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:ExternalSystem/api/accounts');  // Uses Named Credential
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setTimeout(60000);  // 60 seconds
        
        req.setBody(JSON.serialize(new Map<String, Object>{
          'name' => acc.Name,
          'externalId' => acc.External_Id__c
        }));
        
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        if (res.getStatusCode() >= 400) {
          throw new CalloutException('API error: ' + res.getStatusCode());
        }
      }
    } catch (Exception ex) {
      handleError(ex);
    }
  }
  
  private void handleError(Exception ex) {
    // Log and notify
  }
}
```

**Key Points**:
- Named Credentials URL: `callout:CredentialName/path`
- Authentication is handled automatically
- Credentials are encrypted in Salesforce
- No need to include auth header manually

---

## External Credentials (OAuth Integration)

External Credentials are an evolution of Named Credentials. Use these for OAuth flows.

### Setup Steps

1. Setup > Integrations > External Credentials
2. Click New
3. **Name**: `Google_Contacts_Credential`
4. **Auth Type**: `OAuth 2.0`
5. **Principal Type**: Named Principal (all users use the same auth) OR User Principal (each user authorizes)
6. **Principal**: Create or select
7. **Auth Provider**: Select from list (or create custom)
8. Save

### In Apex Code

```apex
public class SyncGoogleContactsQueueable implements Queueable, Database.AllowsCallouts {
  public void execute(QueueableContext context) {
    HttpRequest req = new HttpRequest();
    req.setEndpoint('callout:Google_Contacts_Credential/contacts/v3/people/me/connections');
    req.setMethod('GET');
    req.setHeader('Accept', 'application/json');
    
    Http http = new Http();
    HttpResponse res = http.send(req);
    
    if (res.getStatusCode() == 200) {
      // Process contacts
    }
  }
}
```

**Named Principal vs User Principal**:
- **Named Principal**: One auth for all users (service account). Used for system integrations.
- **User Principal**: Each user authenticates (OAuth redirect). Used for user-specific integrations.

---

## Remote Site Settings & CSP Trusted Sites

If Named Credentials don't work or you need to bypass some settings, configure these.

### Remote Site Settings

Setup > Integrations > Remote Site Settings

- **Remote Site Name**: `Google_API`
- **Remote Site URL**: `https://www.googleapis.com`
- **Disable Protocol Security**: False (keep enabled)

### CSP Trusted Sites (for client-side callouts in LWC)

Setup > Security > CSP Trusted Sites

- **Trusted Site Name**: `Google_API`
- **Trusted Site URL**: `https://www.googleapis.com`
- **CSP Directive**: `connect-src`

**Note**: Remote Site Settings are for Apex. CSP Trusted Sites are for browser/LWC.

---

## HTTP Request Patterns

### Basic Request

```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('https://api.example.com/users');
req.setMethod('GET');
req.setHeader('Authorization', 'Bearer token');
req.setHeader('Content-Type', 'application/json');
req.setTimeout(60000);  // 60 seconds

Http http = new Http();
HttpResponse res = http.send(req);

if (res.getStatusCode() == 200) {
  String body = res.getBody();
  // Parse JSON
}
```

### POST with JSON Body

```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('https://api.example.com/accounts');
req.setMethod('POST');
req.setHeader('Content-Type', 'application/json');

Map<String, Object> payload = new Map<String, Object>{
  'name' => 'Acme Corp',
  'industry' => 'Technology'
};

req.setBody(JSON.serialize(payload));

Http http = new Http();
HttpResponse res = http.send(req);

Integer statusCode = res.getStatusCode();
String responseBody = res.getBody();
```

### Handling Responses

```apex
public class ApiResponse {
  public Integer statusCode;
  public String body;
  public Map<String, Object> parsedBody;
  
  public ApiResponse(HttpResponse res) {
    this.statusCode = res.getStatusCode();
    this.body = res.getBody();
    
    if (statusCode >= 200 && statusCode < 300) {
      this.parsedBody = (Map<String, Object>) JSON.deserializeUntyped(body);
    }
  }
  
  public Boolean isSuccess() {
    return statusCode >= 200 && statusCode < 300;
  }
  
  public Boolean isRetryable() {
    return statusCode >= 500 || statusCode == 429 || statusCode == 408;
  }
}
```

---

## Retry Logic: Exponential Backoff

Auto-retry only on network/timeout errors, not on auth/validation errors.

```apex
public class UpdateExternalSystemQueueable implements Queueable, Database.AllowsCallouts {
  private List<Account> accounts;
  private Integer retryCount = 0;
  private static final Integer MAX_RETRIES = 3;
  
  public UpdateExternalSystemQueueable(List<Account> accounts) {
    this.accounts = accounts;
  }
  
  public UpdateExternalSystemQueueable(List<Account> accounts, Integer retryCount) {
    this.accounts = accounts;
    this.retryCount = retryCount;
  }
  
  public void execute(QueueableContext context) {
    try {
      callExternalSystem();
    } catch (Exception ex) {
      handleError(ex, context);
    }
  }
  
  private void callExternalSystem() {
    HttpRequest req = new HttpRequest();
    req.setEndpoint('callout:ExternalSystem/api/accounts');
    req.setMethod('POST');
    req.setHeader('Content-Type', 'application/json');
    req.setTimeout(60000);
    req.setBody(JSON.serialize(accounts));
    
    Http http = new Http();
    HttpResponse res = http.send(req);
    
    if (res.getStatusCode() >= 400) {
      throw new CalloutException('API error: ' + res.getStatusCode() + ' ' + res.getBody());
    }
  }
  
  private void handleError(Exception ex, QueueableContext context) {
    Boolean isRetryable = isRetryableError(ex.getMessage());
    Boolean hasRetries = retryCount < MAX_RETRIES;
    
    if (isRetryable && hasRetries) {
      Integer delay = (int) Math.pow(5, retryCount + 1);  // 5, 25, 125 seconds
      System.enqueueJob(new UpdateExternalSystemQueueable(accounts, retryCount + 1),
                        System.now().addSeconds(delay));
    } else {
      logPermanentError(ex);
      notifyAdmin(ex);
    }
  }
  
  private Boolean isRetryableError(String message) {
    return message.contains('timeout') || 
           message.contains('connection') || 
           message.contains('503') || 
           message.contains('429') ||
           message.contains('Connect');
  }
  
  private void logPermanentError(Exception ex) {
    ErrorLog__c log = new ErrorLog__c();
    log.Message__c = ex.getMessage();
    log.Stack_Trace__c = ex.getStackTraceString();
    log.Type__c = 'Integration Error';
    log.Retry_Count__c = retryCount;
    insert log;
  }
  
  private void notifyAdmin(Exception ex) {
    Messaging.SingleEmailMessage msg = new Messaging.SingleEmailMessage();
    msg.setToAddresses(new String[]{'admin@company.com'});
    msg.setSubject('Integration Failure: External System Update Failed');
    msg.setPlainTextBody('Permanent failure after ' + retryCount + ' retries.\n\n' + 
                         'Error: ' + ex.getMessage() + '\n\n' + 
                         'Stack: ' + ex.getStackTraceString());
    Messaging.sendEmail(new Messaging.SingleEmailMessage[]{msg});
  }
}
```

**Exponential Backoff**:
- Retry 1: 5 seconds
- Retry 2: 25 seconds
- Retry 3: 125 seconds

This prevents overwhelming an overloaded API.

---

## Timeout Handling

Salesforce has a 120-second timeout for all HTTP callouts.

```apex
private static final Integer TIMEOUT_MS = 60000;  // 60 seconds (leaves buffer)

public void execute(QueueableContext context) {
  HttpRequest req = new HttpRequest();
  req.setTimeout(TIMEOUT_MS);
  
  try {
    Http http = new Http();
    HttpResponse res = http.send(req);
  } catch (System.CalloutException ex) {
    if (ex.getMessage().contains('Timeout')) {
      // Retry (timeout is retryable)
      handleTimeoutError(ex);
    }
  }
}
```

**Best Practice**: Set timeout to 60 seconds max. If API needs longer, split the call into multiple requests.

---

## JSON Serialization & Deserialization

### Safe Serialization

```apex
// ❌ Risky (can throw null reference)
String json = JSON.serialize(payload);

// ✅ Safe
String json = JSON.serialize(payload);  // Salesforce handles nulls

// For custom handling:
Map<String, Object> payload = new Map<String, Object>();
payload.put('name', acc.Name);
payload.put('externalId', acc.External_Id__c != null ? acc.External_Id__c : '');
String json = JSON.serialize(payload);
```

### Safe Deserialization

```apex
// ❌ Risky (throws exception if JSON doesn't match)
ExternalUser user = (ExternalUser) JSON.deserialize(body, ExternalUser.class);

// ✅ Safe
try {
  ExternalUser user = (ExternalUser) JSON.deserialize(body, ExternalUser.class);
} catch (Exception ex) {
  System.debug('JSON deserialization failed: ' + ex.getMessage());
  // Handle gracefully
}

// For untyped JSON:
Map<String, Object> result = (Map<String, Object>) JSON.deserializeUntyped(body);
String name = (String) result.get('name');  // Cast to correct type
```

### Wrapper Classes

```apex
public class ExternalAccountPayload {
  public String name;
  public String externalId;
  public String industry;
  
  public ExternalAccountPayload() {}
  
  public ExternalAccountPayload(Account acc) {
    this.name = acc.Name;
    this.externalId = acc.External_Id__c;
    this.industry = acc.Industry;
  }
}

// Usage
String json = JSON.serialize(new ExternalAccountPayload(acc));
```

---

## Rate Limiting & Backpressure

APIs often have rate limits (e.g., 100 calls per minute).

### Pattern 1: Batching

```apex
public void execute(QueueableContext context) {
  List<Account> batch = new List<Account>();
  
  for (Account acc : accounts) {
    batch.add(acc);
    
    if (batch.size() == 10) {  // Batch of 10
      callBulkApi(batch);
      batch.clear();
    }
  }
  
  if (!batch.isEmpty()) {
    callBulkApi(batch);
  }
}

private void callBulkApi(List<Account> batch) {
  HttpRequest req = new HttpRequest();
  req.setEndpoint('callout:ExternalSystem/api/accounts/bulk');
  req.setBody(JSON.serialize(batch));
  
  Http http = new Http();
  HttpResponse res = http.send(req);
}
```

### Pattern 2: Queue Depth Check

```apex
public void execute(QueueableContext context) {
  AsyncApexJob[] jobs = [SELECT Id FROM AsyncApexJob 
                         WHERE JobType = 'Queueable' AND Status = 'Queued' LIMIT 1];
  
  if (jobs.size() < 5) {  // Max 5 queued jobs
    callExternalSystem();
    System.enqueueJob(new UpdateExternalSystemQueueable(...));
  } else {
    System.debug('Queue is full, retrying later');
    System.enqueueJob(new UpdateExternalSystemQueueable(...));
  }
}
```

---

## Error Handling: Common Patterns

### Pattern 1: Graceful Degradation

```apex
try {
  callExternalSystem();
} catch (Exception ex) {
  System.debug('External call failed, continuing: ' + ex.getMessage());
  // Continue processing local records
  // External sync will retry on next scheduled job
}
```

### Pattern 2: Deadletter Queue

```apex
private void handleError(Exception ex) {
  DeadLetterQueue__c dlq = new DeadLetterQueue__c();
  dlq.Account_Id__c = account.Id;
  dlq.Error_Message__c = ex.getMessage();
  dlq.Attempted_At__c = System.now();
  dlq.Status__c = 'Pending Retry';
  insert dlq;
  
  // Later, a scheduled job processes deadletter queue
}
```

### Pattern 3: Circuit Breaker

```apex
private Boolean checkCircuit() {
  ErrorMetric__c metric = [SELECT Count__c FROM ErrorMetric__c WHERE Type__c = 'Integration' LIMIT 1];
  
  if (metric.Count__c > 10) {  // More than 10 errors in last hour
    System.debug('Circuit open: Too many failures. Blocking new calls.');
    return false;
  }
  
  return true;
}
```

---

## Testing Integrations

```apex
@IsTest
private class UpdateExternalSystemQueueableTest {
  @IsTest
  static void testSuccessfulCallout() {
    Account acc = new Account(Name = 'Test Acme', External_Id__c = 'EXT123');
    insert acc;
    
    Test.startTest();
    
    // Mock the HTTP callout
    Test.setMock(HttpCalloutMock.class, new ExternalSystemMock(200, '{"success": true}'));
    
    System.enqueueJob(new UpdateExternalSystemQueueable(new List<Account>{acc}));
    
    Test.stopTest();
    
    // Verify (no exception thrown)
  }
  
  @IsTest
  static void testRetryOnTimeout() {
    Account acc = new Account(Name = 'Test Acme', External_Id__c = 'EXT123');
    insert acc;
    
    Test.startTest();
    
    // Mock a timeout
    Test.setMock(HttpCalloutMock.class, new TimeoutMock());
    
    System.enqueueJob(new UpdateExternalSystemQueueable(new List<Account>{acc}, 0));
    
    Test.stopTest();
    
    // Verify (queueable job enqueued for retry)
  }
}

// Mock class
global class ExternalSystemMock implements HttpCalloutMock {
  private Integer statusCode;
  private String body;
  
  global ExternalSystemMock(Integer statusCode, String body) {
    this.statusCode = statusCode;
    this.body = body;
  }
  
  global HttpResponse respond(HttpRequest req) {
    HttpResponse res = new HttpResponse();
    res.setStatusCode(statusCode);
    res.setBody(body);
    return res;
  }
}
```

---

## Checklist: Integration Ready for Production

- ✅ No callouts in trigger context (using Queueable)
- ✅ Named Credentials configured and tested
- ✅ Timeout set to <= 60 seconds
- ✅ Retry logic for network errors only
- ✅ Error logging and admin notifications
- ✅ JSON serialization handles nulls
- ✅ Deserialization wrapped in try-catch
- ✅ Rate limiting strategy in place
- ✅ Bulk tested (200+ accounts)
- ✅ All HTTP mocks in tests (no real calls)

