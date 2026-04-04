---
name: sf-integration
description: >
  Salesforce integration skill. Covers Named Credentials, External Credentials, outbound
  REST/SOAP callouts, inbound REST APIs (RestResource), External Services, JSON parsing,
  idempotency, retry design, error handling, callout security, mTLS, Composite API,
  Collections API, API versioning, and rate limiting. Apply for any code involving HTTP
  callouts, external system connectivity, or inbound API endpoints. API 62.0.
allowed-tools: Bash(sf project deploy* --target-org <your-org-alias>*), Bash(sf project retrieve*)
---

# Salesforce Integration Skill

**Named Credentials, External Credentials, outbound REST/SOAP callouts, inbound REST APIs, idempotency, retry design, error handling, Composite API, and rate limiting.**

## Core Principle

Integrations are the #1 source of data corruption and governor limit breaches. Every callout must be async (never in trigger context), idempotent (safe to retry), and secured via Named Credentials.

> **Order of Execution Note**
> Triggers fire at steps 3 (before) and 7 (after) of Salesforce's 14-step save sequence. Both run inside a synchronous database transaction. Callouts are blocked at these steps because an uncommitted transaction and an outbound HTTP call together could leave data in an inconsistent state if either side fails.
> The fix is always Queueable with `Database.AllowsCallouts`. The trigger fires, enqueues the job, the transaction commits, then the Queueable runs the callout outside trigger context.

---

## Authentication -- Named Credentials + External Credentials (API 62.0)

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
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:OpenWeatherMap_NC/weather?q=London&units=metric');
req.setMethod('GET');
req.setTimeout(10000);
```

---

## Callout Rules

1. **Never in trigger context** -- throws `System.CalloutException`
2. **Always use Queueable** for callouts triggered by DML events
3. **Always set a timeout** -- default is 10s; max is 120s
4. **Always use HTTPS** -- HTTP blocked in production
5. **Always use Named Credentials** -- never hardcode URLs or API keys
6. **Design for idempotency** -- external systems retry

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

// BEST - typed wrapper for known schemas
MyResponseWrapper resp = (MyResponseWrapper) JSON.deserialize(body, MyResponseWrapper.class);
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

---

## Idempotency -- Design for Retry

```apex
// upsert with External ID -- same data twice = same record
Database.upsert(incomingRecords, External_ID__c.class, false);

// deduplicate before DML
Map<String, City_Weather__c> unique = new Map<String, City_Weather__c>();
for (City_Weather__c rec : incoming) {
    unique.put(rec.City_Name__c, rec);
}
Database.upsert(unique.values(), City_Weather__c.City_Name__c, false);
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

- [ ] Named Credential and External Credential metadata exist
- [ ] All callouts are in Queueable or Batch -- never in trigger or synchronous context
- [ ] Timeout set on every `HttpRequest`
- [ ] HTTPS only
- [ ] JSON parsing uses typed wrapper classes or null-safe casts
- [ ] Upsert with External ID for idempotency on all inbound data
- [ ] `allOrNone=false` for partial success
- [ ] Error path publishes Platform Event or stores failure for retry
- [ ] `@RestResource` classes use `with sharing` and validate all inputs
- [ ] No credentials, tokens, or org-specific IDs in code
- [ ] 429 rate-limit responses handled with Retry-After logic
- [ ] Inbound API URL includes version: `/v1/`, `/v2/`

---

## Sources

- [Salesforce Integration Patterns and Practices](https://developer.salesforce.com/docs/atlas.en-us.integration_patterns_and_practices.meta/integration_patterns_and_practices/integ_pat_intro_overview.htm)
- [Apex Callouts](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts.htm)
- [Named Credentials](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts_named_credentials.htm)
- [Apex REST Resources](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_rest.htm)
- [Composite API](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_composite_composite.htm)
- [Bulk API 2.0](https://developer.salesforce.com/docs/atlas.en-us.api_asynch.meta/api_asynch/asynch_api_intro.htm)
- [Well-Architected: Scalable](https://architect.salesforce.com/well-architected/adaptable/scalable)
