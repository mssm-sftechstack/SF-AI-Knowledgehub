# SALESFORCE PROJECT — AI AGENT RULES

These are the development rules for this Salesforce DX project. Any AI tool that reads `AGENTS.md` should apply all rules below to every file it generates or modifies. Tools that don't read this file automatically can use its contents as a system prompt at the start of a session.

---

## Org Context

<!-- FILL IN: replace placeholders with your org details -->
- Org Alias: <your-org-alias>
- API Version: 62.0
- Instance URL: <your-instance-url>
- Deploy: `sf project deploy start --source-dir force-app --target-org <your-org-alias>`
- Retrieve: `sf project retrieve start --manifest manifest/package.xml --target-org <your-org-alias>`

---

## Governor Limit Rules

Salesforce enforces hard transaction limits: 100 SOQL queries, 150 DML statements, 6MB heap, 10 seconds CPU per transaction.

| Rule | Why |
|---|---|
| Never put SOQL or DML inside a for loop | 200 records = 200 queries = instant limit hit |
| Bulkify every method for 200+ records | Triggers fire with up to 200 records per execution |
| Use Map<Id, SObject> for lookups inside loops | Avoids repeated queries for related data |
| Use Database.executeBatch() for large data volumes | Batch Apex gets 200 records per chunk with its own limits |

---

## Security Rules

| Rule | Correct Pattern |
|---|---|
| Field-level security in SOQL | Use `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()` |
| Sharing model | `with sharing` on all Apex classes; `inherited sharing` on Selectors |
| No hardcoded IDs | Resolve via `getRecordTypeInfosByDeveloperName()` or SOQL at runtime |
| No callouts in triggers | Use `Queueable implements Database.AllowsCallouts` |
| No `seeAllData=true` | Use `@IsTest(SeeAllData=false)` and create test data explicitly |
| LWC XSS | Never set `innerHTML` with user-supplied values. Use `textContent`. |
| Permission Sets | Add all new access via Permission Sets, never directly to Profiles |
| Picklist values | Read `.field-meta.xml` before setting a restricted picklist. Use exact `<fullName>` strings. |

---

## Architecture Pattern

All Apex follows this four-layer structure:

```
Trigger (one per object — zero logic, delegates to handler)
  → TriggerHandler (routes by before/after, insert/update/delete)
    → Service (business logic, no SOQL)
      → Selector (all SOQL, inherited sharing)
```

- Never create a second trigger on an object that already has one. Extend the handler.
- Service classes contain no SOQL. All queries go through a Selector.
- Selector classes use `inherited sharing` so the caller's sharing context applies.

**Example trigger (the complete correct pattern):**

```apex
trigger OpportunityTrigger on Opportunity (
    before insert, before update, after insert, after update, before delete, after delete
) {
    new OpportunityTriggerHandler().run();
}
```

---

## Testing Rules

Every new Apex class requires a paired test class. All test classes must meet these criteria:

- `@IsTest(SeeAllData=false)` always. Never `seeAllData=true`.
- 90%+ code coverage minimum.
- At least one bulk test method with 200 records.
- `System.runAs()` for any test calling code that uses `Security.stripInaccessible()` or `WITH USER_MODE`. System Admin bypasses FLS silently, giving false passes.
- Assert specific field values with `System.assertEquals(expected, actual, 'descriptive message')`.
- Cover the positive path, negative path (invalid input, missing required fields), and bulk path.

---

## Metadata Rules

- API version: 62.0 in all metadata files.
- Object metadata: `enableBulkApi`, `enableSharing`, and `enableStreamingApi` must ALL be the same boolean value.
- Custom Tabs must exist in the org before any Permission Set references them.
- Permission Sets must include `fieldPermissions` for every NON-REQUIRED custom field on new custom objects. Required fields must be excluded from `fieldPermissions` or deployment fails.
- Remote Site Setting URLs must match the org's Instance URL character-by-character.

---

## Deployment Order

Deploy metadata in this order when multiple types are involved:

1. Custom Objects and Fields
2. Custom Metadata Types
3. Custom Settings
4. Custom Tabs
5. Permission Sets and Custom Permissions
6. Apex Classes (non-test)
7. Apex Test Classes
8. Triggers
9. Remote Site Settings and CSP Trusted Sites
10. Named Credentials and External Credentials
11. Flows
12. LWC and Aura Components
13. FlexiPages
14. Page Layouts and Compact Layouts
15. Applications
16. Profiles and Permission Set Assignments

Always validate before deploying:
```bash
sf project deploy validate --source-dir force-app --target-org <your-org-alias>
```

Never deploy without explicit user confirmation.

---

## CLI and Tooling

- Use `sf` CLI v2 commands only. Never `sfdx` (deprecated).
- Use `sf org login web --alias <alias>` not `sfdx force:auth:web:login`.
- Use `sf project deploy start` not `sfdx force:source:deploy`.

---

## Code Style

- No AI-pattern comments: never write "This ensures that...", "It's important to note...", "is responsible for...", "This method handles...".
- No em dashes in comments or documentation.
- Class names: PascalCase. Methods: camelCase. Test classes: `ClassNameTest`. Selectors: `ObjectNameSelector`.
- Comments in plain English. Short, direct, written like a human left a note.
