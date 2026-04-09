# Enterprise Architecture Mandates for AI Generation

**CRITICAL DIRECTIVE FOR AI:** You are generating code for Salesforce, a strictly governed multi-tenant environment. You are not writing local Node.js or Python scripts. If your generated code is inefficient, it will not just run slowly. Salesforce will forcefully terminate the transaction and crash the user's operation.

Every piece of Apex, LWC, or Integration code you generate must strictly adhere to these 5 architectural pillars. For numerical limits, refer to the [Multitenant & Governor Limits Guide](./MULTITENANT_AND_GOVERNOR_LIMITS.md).

## 1. The Bulkification Mandate
Code must be designed to process 1 record or 10,000 records with the exact same number of SOQL queries and DML statements. 
* NEVER query or perform DML inside a loop.
* ALWAYS use Maps (`Map<Id, SObject>`) to relate parent and child records in memory.
* For handling 50,000+ records, see the [Large Data Volume Constraints](./LARGE_DATA_VOLUME_CONSTRAINTS.md).

## 2. The Context-Aware Mandate
Salesforce executes code in specific contexts (System Context vs. User Context).
* Apex runs in System Context by default, bypassing user permissions. You MUST explicitly enforce sharing rules using `with sharing` on all classes unless architecturally justified. See the [Zero-Trust Security Model](./ZERO_TRUST_SECURITY_MODEL.md).
* LWC runs in User Context. You MUST NOT leak restricted data to the client-side via unrestricted Apex controllers. See [LWC Security & Limits](./LWC_SECURITY_AND_LIMITS.md).

## 3. The Asynchronous Offloading Mandate
Synchronous Apex limits CPU time to 10,000 milliseconds.
* If a process involves callouts, heavy XML/JSON parsing, or complex data transformations, you MUST offload it using Queueable Apex or Platform Events. See [Integration & Callout Constraints](./INTEGRATION_AND_CALLOUT_CONSTRAINTS.md).
* Do not lock the main execution thread.

## 4. The "No Hardcoding" Mandate
Metadata IDs (Record Types, Queues, Profiles) change between sandboxes and production.
* NEVER hardcode `012...` or `00G...` IDs in your code.
* ALWAYS use `Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName().get('DevName').getRecordTypeId()` or Custom Metadata.

## 5. The Automation Hierarchy Mandate
Apex is the tool of last resort for same-record updates.
* Before generating an Apex Trigger, determine if a Before-Save Fast Field Update Flow can handle the requirement. Review the [Order of Execution](./ORDER_OF_EXECUTION.md) and [Flow Constraints](./FLOW_AND_AUTOMATION_CONSTRAINTS.md) to prevent recursion.