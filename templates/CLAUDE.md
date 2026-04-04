# SALESFORCE PROJECT — CLAUDE CODE CONFIGURATION

## Org Context

<!-- FILL IN: replace the placeholders below with your org details -->
- Default Org Alias: <your-org-alias>
- API Version: 62.0
- Instance URL: <your-instance-url>
- Deploy command: sf project deploy start --source-dir force-app --target-org <your-org-alias>
- Retrieve command: sf project retrieve start --manifest manifest/package.xml --target-org <your-org-alias>

## Hard Rules — Apply Without Exception

1. Always ask for confirmation before running any deployment command.
2. Only deploy to the org alias defined above. All other targets are blocked.
3. One trigger per Salesforce object. Extend the handler, never create a second trigger.
4. Never put SOQL or DML inside a for loop.
5. Never use hardcoded IDs, org IDs, Record Type IDs, or credentials in code.
6. Never use seeAllData=true in test classes.
7. Always use `with sharing` on Apex classes. Document any exceptions with a comment explaining why.
8. Always run a security review before any deployment.
9. Always write a test class for every new Apex class. 90%+ coverage minimum.
10. Always use current `sf` CLI commands. Never use deprecated `sfdx` syntax.
11. Always bulkify all Apex. Design for 200+ records minimum.
12. Always use WITH SECURITY_ENFORCED or Security.stripInaccessible() in SOQL.
13. Always resolve Record Type IDs dynamically using getRecordTypeInfosByDeveloperName().
14. Never make callouts from trigger context. Use Queueable with Database.AllowsCallouts.
15. Never write AI-pattern comments: "This ensures that...", "It's important to note...", "is responsible for..."
16. Always read every restricted picklist field's .field-meta.xml before writing code that sets it. Never assume picklist values.
17. Object metadata enableBulkApi, enableSharing, enableStreamingApi must ALL be the same value. Mixing true/false causes deploy failure.
18. Always use System.runAs() in tests that call methods using Security.stripInaccessible() or WITH USER_MODE.
19. Never deploy without explicit user confirmation.
20. Always score Apex (120+/150), LWC (132+/165), Flow (88+/110) before marking ready to ship.
21. Always include fieldPermissions in a PermissionSet for every NON-REQUIRED custom field on every new custom object.
22. Always verify Remote Site Setting URLs character-by-character against the Instance URL above.

## Deployment Order for Complex Changes

Note: Platform Cache Partitions must be created in Setup UI. They're not deployable via metadata.

1. Custom Objects and Fields
2. Custom Metadata Types
3. Custom Settings
4. Custom Tabs (must exist in org BEFORE any Permission Set references them)
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

## Code Style

- Class names: PascalCase
- Method names: camelCase
- Test classes: ClassNameTest (e.g. AccountServiceTest)
- Selector classes: ObjectNameSelector (e.g. AccountSelector)
- No em dashes in code comments
- Short, plain English comments. Write like a human left a note, not like documentation.
- No AI-pattern filler phrases in any generated text

## Architecture Pattern

All Apex follows this layer structure:

```
Trigger (delegate only, no logic)
  → TriggerHandler (route by context: before/after, insert/update)
    → Service (business logic, no SOQL)
      → Selector (all SOQL, inherited sharing)
```

- Triggers contain zero logic. One line calling the handler.
- Service classes have no SOQL. All queries go through Selector classes.
- Selector classes use `inherited sharing`.
- Service and Handler classes use `with sharing`.
