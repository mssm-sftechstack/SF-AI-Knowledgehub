# Salesforce Integration Skill

**Named Credentials, External Credentials, outbound REST/SOAP callouts, inbound REST APIs, idempotency, retry design, error handling, Composite API, and rate limiting.**

## Core Principle

Integrations are the #1 source of data corruption and governor limit breaches. Every callout must be async (never in trigger context), idempotent (safe to retry), and secured via Named Credentials.

**Reference:** [Salesforce Integration Patterns and Practices](https://developer.salesforce.com/docs/atlas.en-us.integration_patterns_and_practices.meta/integration_patterns_and_practices/integ_pat_intro_overview.htm)

> **Order of Execution Note**
> Triggers fire at steps 3 (before) and 7 (after) of Salesforce's 14-step save sequence. Both run inside a synchronous database transaction. Callouts are blocked at these steps because an uncommitted transaction and an outbound HTTP call together could leave data in an inconsistent state if either side fails.
> The fix is always Queueable with `Database.AllowsCallouts`. The trigger fires, enqueues the job, the transaction commits, then the Queueable runs the callout outside trigger context.

---

## Authentication -- Named Credentials + External Credentials (API 62.0)

API 62.0 splits authentication from the endpoint. Use this pattern for all new integrations:

```
External Credential  ->  defines the auth protocol (OAuth 2.0, Custom, API Key, etc.)
Named Credential     ->  defines the endpoint URL, references the External Credential
Principal            ->  per-org or per-user credential storage
```

```xml
<!-- ExternalCredential: OpenWeatherMap_EC.externalCredential-meta.xml -->
<ExternalCredential>
    <authenticationProtocol>Custom</authenticationProtocol>
    <label>OpenWeatherMap EC</label>
    <parameters>
        <parameterName>API_Key</parameterName>
        <parameterType>AuthQueryStringParam</parameterType>
        <sequenceNumber>1</sequenceNumber>
    </parameters>
    <principals>
        <parameterName>API_Key</parameterName>
        <principalType>NamedPrincipal</principalType>
        <sequenceNumber>1</sequenceNumber>
    </principals>
</ExternalCredential>

<!-- NamedCredential: OpenWeatherMap_NC.namedCredential-meta.xml -->
<NamedCredential>
    <label>OpenWeatherMap NC</label>
    <endpoint>https://api.openweathermap.org/data/2.5</endpoint>
    <externalCredential>OpenWeatherMap_EC</externalCredential>
    <allowMergeFieldsInBody>false</allowMergeFieldsInBody>
    <allowMergeFieldsInHeader>false</allowMergeFieldsInHeader>
    <generateAuthorizationHeader>false</generateAuthorizationHeader>
</NamedCredential>
```

```apex
// callout using Named Credential -- no credentials in code
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:OpenWeatherMap_NC/weather?q=London&units=metric');
req.setMethod('GET');
req.setTimeout(10000);
```

**Reference:** [Named Credentials](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts_named_credentials.htm)

---

## Callout Rules

1. **Never in trigger context** -- triggers are synchronous DML transactions; callout throws `System.CalloutException`
2. **Always use Queueable** for callouts triggered by DML events
3. **Always set a timeout** -- default is 10s; max is 120s. No timeout means a hung transaction
4. **Always use HTTPS** -- HTTP endpoints are blocked in production
5. **Always use Named Credentials** -- never hardcode URLs, usernames, passwords, or API keys
6. **Design for idempotency** -- external systems retry; your handler must produce the same result on duplicates

---

## Outbound REST Callout Pattern

```apex
public with sharing class WeatherService {

    public static WeatherData fetchWeather(String city) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:OpenWeatherMap_NC/weather?q='
            + EncodingUtil.urlEncode(city, 'UTF-8') + '&units=metric');
        req.setMethod('GET');
        req.setTimeout(10000);

        HttpResponse res = new Http().send(req);

        if (res.getStatusCode() == 200) {
            return parseResponse(res.getBody());
        } else if (res.getStatusCode() == 404) {
            throw new WeatherApiException('City not found: ' + city);
        } else if (res.getStatusCode() == 429) {
            throw new WeatherApiException('Rate limited: retry after ' + res.getHeader('Retry-After') + 's');
        } else {
            throw new WeatherApiException('API error ' + res.getStatusCode());
        }
    }

    private static WeatherData parseResponse(String body) {
        Map<String, Object> root = (Map<String, Object>) JSON.deserializeUntyped(body);
        Map<String, Object> main = (Map<String, Object>) root.get('main');

        WeatherData data    = new WeatherData();
        data.temperature    = (Decimal) main.get('temp');
        data.humidity       = Integer.valueOf(String.valueOf(main.get('humidity')));
        data.cityName       = (String) root.get('name');
        return data;
    }

    public class WeatherData {
        public String  cityName;
        public Decimal temperature;
        public Integer humidity;
    }

    public class WeatherApiException extends Exception {}
}
```

### Safe JSON Parsing

```apex
// WRONG - ClassCastException if key missing or wrong type
Integer temp = (Integer) root.get('temp');

// CORRECT - null-safe with type check
Object rawTemp = root.get('temp');
Decimal temp = rawTemp instanceof Decimal ? (Decimal) rawTemp
             : rawTemp instanceof Integer ? Decimal.valueOf((Integer) rawTemp)
             : null;

// BEST - typed wrapper class for known response schemas
MyResponseWrapper resp = (MyResponseWrapper) JSON.deserialize(body, MyResponseWrapper.class);
```

---

## Outbound SOAP Callout Pattern

```apex
// generate Apex stub from WSDL
// sf apex generate fromwsdl --wsdl path/to/service.wsdl --output-dir force-app/main/default/classes

MyService.MyServicePort stub = new MyService.MyServicePort();
stub.endpoint_x = 'callout:MySOAP_NC/endpoint';
stub.timeout_x  = 10000;
MyService.ResponseType result = stub.callMethod(param1, param2);
```

---

## Certificate-Based Authentication (mTLS)

Use mutual TLS when the external system requires client certificate auth (banking, healthcare, government):

```
Setup -> Certificate and Key Management -> Create Self-Signed Certificate

Then reference in Named Credential:
  Authentication Protocol: No Authentication (mTLS handles it at transport layer)
  Certificate: [select the certificate you uploaded]
```

Key rules:
- Certificate expiry is a production incident -- set a calendar reminder 60 days before expiry
- Store CA-signed certificates for external system trust
- Rotate certificates before expiry; deploy the new Named Credential before the old certificate expires

---

## Salesforce REST API -- Composite and Collections

### Composite API -- Multiple Operations in One HTTP Request

```bash
POST /services/data/v62.0/composite
{
  "allOrNone": true,
  "compositeRequest": [
    {
      "method": "POST",
      "url": "/services/data/v62.0/sobjects/Account",
      "referenceId": "newAccount",
      "body": { "Name": "Acme Corp", "BillingCountry": "US" }
    },
    {
      "method": "POST",
      "url": "/services/data/v62.0/sobjects/Contact",
      "referenceId": "newContact",
      "body": {
        "FirstName": "Jane",
        "LastName": "Doe",
        "AccountId": "@{newAccount.id}"
      }
    }
  ]
}
```

### Collections API -- Bulk DML in One Request (up to 200 records)

```bash
POST /services/data/v62.0/composite/sobjects
{
  "allOrNone": false,
  "records": [
    { "attributes": { "type": "Contact" }, "FirstName": "Alice", "LastName": "Smith" },
    { "attributes": { "type": "Contact" }, "FirstName": "Bob",   "LastName": "Jones" }
  ]
}
```

---

## Rate Limiting and Retry-After Handling

| Limit | Value |
|---|---|
| API calls per org per 24h (Enterprise) | 1,000,000 |
| Concurrent API requests per org | 25 |
| Bulk API v2 jobs per 24h | 10,000 |

```apex
if (res.getStatusCode() == 429) {
    String retryAfter = res.getHeader('Retry-After');
    Integer waitSeconds = String.isNotBlank(retryAfter) ? Integer.valueOf(retryAfter) : 60;
    System.enqueueJob(new RetryCalloutJob(payload), waitSeconds);
}
```

---

## Inbound REST API

```apex
@RestResource(urlMapping='/weather/v1/sync/*')
global with sharing class WeatherSyncResource {

    @HttpPost
    global static SyncResponse doPost() {
        RestRequest  req  = RestContext.request;
        RestResponse resp = RestContext.response;

        try {
            WeatherPayload payload = (WeatherPayload) JSON.deserialize(
                req.requestBody.toString(), WeatherPayload.class
            );

            if (String.isBlank(payload.cityName)) {
                resp.statusCode = 400;
                return new SyncResponse(false, 'cityName is required');
            }

            // idempotent upsert -- same payload twice = same record
            City_Weather__c rec = new City_Weather__c(
                City_Name__c   = payload.cityName,
                Temperature__c = payload.temperature,
                Last_Synced__c = System.now()
            );
            Database.upsert(rec, City_Weather__c.City_Name__c, false);

            resp.statusCode = 200;
            return new SyncResponse(true, 'Synced');

        } catch (JSONException e) {
            resp.statusCode = 400;
            return new SyncResponse(false, 'Invalid JSON');
        } catch (Exception e) {
            resp.statusCode = 500;
            return new SyncResponse(false, 'Server error');
        }
    }

    global class WeatherPayload {
        public String  cityName;
        public Decimal temperature;
    }

    global class SyncResponse {
        public Boolean success;
        public String  message;
        public SyncResponse(Boolean s, String m) { success = s; message = m; }
    }
}
```

Inbound security rules:
- Always `global with sharing` on `@RestResource` classes
- Validate all input before DML -- never trust inbound payloads
- Never expose stack traces in error responses
- Version your inbound API in the URL: `/v1/`, `/v2/`

---

## Idempotency -- Design for Retry

```apex
// upsert with External ID -- same data twice = same record, not two records
Database.upsert(incomingRecords, External_ID__c.class, false);

// deduplicate before DML when multiple records could have the same key
Map<String, City_Weather__c> unique = new Map<String, City_Weather__c>();
for (City_Weather__c rec : incoming) {
    unique.put(rec.City_Name__c, rec);
}
Database.upsert(unique.values(), City_Weather__c.City_Name__c, false);
```

---

## Error Handling and Retry Design

```apex
// partial success -- don't let one bad record fail all
Database.UpsertResult[] results = Database.upsert(records, External_ID__c.class, false);
List<String> errors = new List<String>();
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        for (Database.Error err : results[i].getErrors()) {
            errors.add('Record ' + i + ': ' + err.getMessage());
        }
    }
}
if (!errors.isEmpty()) {
    // publish a Platform Event to the retry queue
}
```

---

## Integration Pattern Selector

| Scenario | Pattern |
|---|---|
| DML event triggers callout | Queueable + `Database.AllowsCallouts` |
| Scheduled sync of external data | Schedulable -> Batch Apex |
| Receive inbound data from external | `@RestResource` with `@HttpPost` |
| Real-time decoupled event | Platform Event |
| Simple REST API with OpenAPI spec | External Services + Flow |
| Legacy SOAP system | WSDL2Apex stub via Named Credential |
| Bulk data load (external calls SF) | Bulk API 2.0 |
| Multiple SF ops in single HTTP call | Composite API |
| Batch upsert up to 200 records | Collections API |
| Client certificate auth (mTLS) | Certificate + Named Credential |

---

## Pre-Integration Checklist

- [ ] Named Credential and External Credential metadata exist (no hardcoded endpoints)
- [ ] All callouts are in Queueable or Batch -- never in trigger or synchronous context
- [ ] Timeout set on every `HttpRequest`
- [ ] HTTPS only
- [ ] JSON parsing uses typed wrapper classes or null-safe casts
- [ ] Upsert with External ID for idempotency on all inbound data
- [ ] `Database.upsert/saveResult` with `allOrNone=false` for partial success
- [ ] Error path publishes Platform Event or stores failure for retry
- [ ] `@RestResource` classes use `with sharing` and validate all inputs
- [ ] No credentials, tokens, or org-specific IDs in code
- [ ] 429 rate-limit responses handled with Retry-After logic
- [ ] API version pinned in Named Credential endpoint URL
- [ ] Certificate expiry date tracked (if mTLS)
- [ ] Inbound API URL includes version: `/v1/`, `/v2/`

---

## Sources

- [Salesforce Integration Patterns and Practices](https://developer.salesforce.com/docs/atlas.en-us.integration_patterns_and_practices.meta/integration_patterns_and_practices/integ_pat_intro_overview.htm)
- [Apex Callouts](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts.htm)
- [Named Credentials](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts_named_credentials.htm)
- [External Credentials](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts_external_credentials.htm)
- [Apex REST Resources](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_rest.htm)
- [Composite API](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_composite_composite.htm)
- [Bulk API 2.0](https://developer.salesforce.com/docs/atlas.en-us.api_asynch.meta/api_asynch/asynch_api_intro.htm)
- [Well-Architected: Scalable](https://architect.salesforce.com/well-architected/adaptable/scalable)
