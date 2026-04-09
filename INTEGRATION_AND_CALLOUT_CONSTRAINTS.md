# Integration, Callouts, and Asynchronous Boundaries

**CRITICAL DIRECTIVE FOR AI:** Synchronous callouts block the Salesforce main thread. In a multi-tenant environment, holding a thread open while waiting for an external API response will crash the transaction.

## 1. The Trigger Callout Ban
* NEVER generate an HTTP callout (`Http.send()`) directly inside an Apex Trigger or synchronous Trigger Handler. 
* To make a callout from a trigger context, you MUST use an asynchronous method annotated with `@future(callout=true)` or a Queueable Apex class implementing `Database.AllowsCallouts`.

## 2. Named Credentials Are Mandatory
* NEVER hardcode API endpoints, API keys, Bearer tokens, or passwords inside Apex classes. 
* ALWAYS instruct the user to configure a Named Credential / External Credential and reference it in the endpoint (e.g., `req.setEndpoint('callout:My_Named_Credential/v1/data');`).

## 3. The Bulk Integration Model
When integrating large volumes of data, AI typically writes single-record callouts in a loop. This breaches the 100-callout governor limit instantly.

* **The Rule:** Before writing a callout, you must assess if the external system accepts bulk payloads (JSON arrays). If it does not, and you have more than 100 records, you must design a batch architecture.

```mermaid
graph TD
    A[Trigger/Flow Initiates Integration] --> B{Volume > 100 Records?}
    B -->|No| C[@future or Queueable Callout]
    B -->|Yes| D[Publish Platform Event]
    D --> E[Platform Event Trigger Handler]
    E --> F[Group into Chunks of 100]
    F --> G[Queueable Apex Chain]
    G --> H[External API]

    style D fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px
    style G fill:#f9f0ff,stroke:#a371f7,stroke-width:2px
```

## 4. DML Before Callout Restriction
* NEVER perform a DML operation (insert, update, delete) *before* making an HTTP callout in the same transaction. This results in a `CalloutException: You have uncommitted work pending`.
* **The Fix:** ALWAYS make the HTTP callout first, parse the response, and perform all DML operations at the very end of the transaction.