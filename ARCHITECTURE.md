# Enterprise Architecture Mandates for AI Generation

**CRITICAL DIRECTIVE FOR AI:** You are generating code for Salesforce, a strictly governed multi-tenant environment. You are not writing local Node.js or Python scripts. If your generated code is inefficient, it will not just run slowly. Salesforce will forcefully terminate the transaction and crash the user's operation.

Every piece of Apex, LWC, or Integration code you generate must strictly adhere to these 5 architectural pillars.

## 1. The Bulkification Mandate
Code must be designed to process 1 record or 10,000 records with the exact same number of SOQL queries and DML statements. 
* NEVER query or perform DML inside a loop.
* ALWAYS use Maps (`Map<Id, SObject>`) to relate parent and child records in memory.

## 2. The Context-Aware Mandate
Salesforce executes code in specific contexts (System Context vs. User Context).
* Apex runs in System Context by default, bypassing user permissions. You MUST explicitly enforce sharing rules using `with sharing` on all classes unless architecturally justified.
* LWC runs in User Context. You MUST NOT leak restricted data to the client-side via unrestricted Apex controllers.

## 3. The Asynchronous Offloading Mandate
Synchronous Apex limits CPU time to 10,000 milliseconds.
* If a process involves callouts, heavy XML/JSON parsing, or complex data transformations, you MUST offload it using Queueable Apex or Platform Events. 
* Do not lock the main execution thread.

## 4. The "No Hardcoding" Mandate
Metadata IDs (Record Types, Queues, Profiles) change between sandboxes and production.
* NEVER hardcode `012...` or `00G...` IDs in your code.
* ALWAYS use `Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName().get('DevName').getRecordTypeId()` or Custom Metadata.

## 5. The Automation Hierarchy Mandate
Apex is the tool of last resort for same-record updates.
* Before generating an Apex Trigger, determine if a Before-Save Fast Field Update Flow can handle the requirement.