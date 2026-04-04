# Common Errors Reference

**Root causes and fixes for the Salesforce errors you'll hit most often.**

---

## Deploy Errors

| Error | Root Cause | Fix |
|---|---|---|
| `Cannot deploy to a required field` | `fieldPermissions` in a Permission Set includes a required field | Remove that field from `fieldPermissions`. Required fields don't need it. |
| `Tab not found` in Permission Set | Tab referenced in `tabSettings` doesn't exist in the org yet | Deploy the Tab before the Permission Set |
| `Invalid field: [FieldName]` in LWC | Field not in schema or no FLS access | Check the field API name, check fieldPermissions in the deployed PS |
| `Class not found` in trigger | Trigger handler class wasn't included in the deploy | Add the handler class to the deploy package |
| `Auth provider not found` in Named Credential | Remote Site Setting or AuthProvider isn't deployed | Deploy RSSettings/AuthProvider first |
| `enableBulkApi and enableSharing must match` | Object metadata has mixed true/false values | Set all three (enableBulkApi, enableSharing, enableStreamingApi) to the same value |
| `Dependent class is invalid and needs recompilation` | Apex class references another class that changed | Redeploy the dependent class alongside its dependencies |

---

## Apex Runtime Errors

| Error | Root Cause | Fix |
|---|---|---|
| `System.LimitException: Too many SOQL queries: 101` | SOQL inside a for loop | Move the query outside the loop, use a Map for lookups |
| `System.LimitException: Too many DML statements: 151` | DML inside a for loop | Collect records in a list, single DML after the loop |
| `MIXED_DML_OPERATION` | Inserting a User and a non-setup object in the same transaction | Wrap User insert in `System.runAs()` |
| `System.JSONException: Unexpected character` | Deserialising a null or malformed response body | Null-check `res.getBody()` before calling `JSON.deserializeUntyped()` |
| `Callout from scheduled Apex not supported` | Testing a schedulable via `System.schedule()` + `Test.stopTest()` | Call `execute(null)` directly in your test instead |
| `SObject row was retrieved via SOQL without querying the requested field` | Field used in code wasn't in the SELECT | Add the field to the SOQL query |
| `You have uncommitted work pending` | Making a callout after DML in the same transaction | Move DML before callouts, or use async context |

---

## FLS and Sharing Errors

| Error | Root Cause | Fix |
|---|---|---|
| `Field is not readable` | `WITH SECURITY_ENFORCED` blocking a field the user can't see | Add `fieldPermissions` for that field in the user's Permission Set |
| `No such column` on a custom field | FLS strips the field with `WITH USER_MODE` or `stripInaccessible()` | Add `fieldPermissions` in the PS, or run the test as a user with the PS assigned |
| Record returns empty but exists in org | Sharing rules or OWD restrict visibility | Check OWD, add sharing rules, or use `without sharing` with documented justification |
| `Insufficient access rights on cross-reference id` | User doesn't have access to the parent record | Check sharing on the parent object |

---

## Test Errors

| Error | Root Cause | Fix |
|---|---|---|
| `System.AssertException: Assertion failed` | Test assertion mismatch | Check the expected value. Use `System.assertEquals(expected, actual, 'message')` with a message. |
| `CANNOT_INSERT_UPDATE_ACTIVATE_ENTITY` | Trigger fires during test and throws an error | Check trigger logic for missing null guards on test data |
| `Static variable is null in async context` | Bypass flag set via static variable doesn't survive Queueable | Use Custom Permissions for bypass, not static booleans |
| `Variable does not exist: Trigger.new` | Using `Trigger.new` outside a trigger context | Use the handler constructor pattern to pass `Trigger.new` in |

---

## Agentforce / Invocable Errors

| Error | Root Cause | Fix |
|---|---|---|
| Wrong action fires | Vague `description` on `@InvocableMethod` | Rewrite description with specific trigger phrases ("Use when user says...") |
| Action fires multiple times | Agent retries on ambiguous response | Make the action idempotent (check-then-act) |
| Summary field is blank | `summary` not set in the response class | Set `res.summary` before returning. The agent relies on it. |
| PII in agent response | Summary string includes email/phone data | Reference names and case numbers, not raw contact fields |
