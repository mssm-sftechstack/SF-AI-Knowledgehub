# Deployment Order Reference

**Deploy in this order every time. Each type depends on the types above it.**

---

## The Order

| Step | Metadata Type | Key Dependency |
|---|---|---|
| 1 | Custom Objects and Fields | Nothing. They're the foundation. |
| 2 | Custom Metadata Types | Must exist before Apex or Flows read them. |
| 3 | Custom Settings | Same rule: deploy before the code that reads them. |
| 4 | Custom Tabs | Permission Sets reference tabs. Tabs come first. |
| 5 | Permission Sets and Custom Permissions | Reference fields and tabs. Both must exist. |
| 6 | Apex Classes (non-test) | Compiled against the schema. Objects and fields must exist. |
| 7 | Apex Test Classes | Depend on production Apex. Deploy after step 6. |
| 8 | Triggers | Reference handler classes. Handlers go first. |
| 9 | Remote Site Settings and CSP Trusted Sites | Required before callouts. |
| 10 | Named Credentials and External Credentials | Depend on Remote Site Settings. |
| 11 | Flows | Reference objects, Apex actions, and Permission Sets. All must exist. |
| 12 | LWC and Aura Components | Reference Apex. Apex goes first. |
| 13 | FlexiPages | Reference LWC components. Components go first. |
| 14 | Page Layouts and Compact Layouts | Reference fields. Fields must exist. |
| 15 | Applications | Reference tabs. Tabs must exist. |
| 16 | Profiles and Permission Set Assignments | Reference everything. Always last. |

---

## Common Ordering Mistakes

| Mistake | Error You'll See | Fix |
|---|---|---|
| Permission Set `tabSettings` before the Tab | "Tab not found" reference error | Deploy Tab at step 4, PS at step 5 |
| Named Credential before Remote Site Setting | "Auth provider not found" | Deploy RSSettings at step 9, NC at step 10 |
| LWC before its Apex class | "Apex method not found" compile error | Deploy Apex at step 6, LWC at step 12 |
| Trigger before its handler class | "Class not found" compile error | Handler at step 6, trigger at step 8 |
| Flow before its Permission Set | Flow validation error on activation | PS at step 5, Flow at step 11 |

---

## Things You Can't Deploy via Metadata

Create these manually in Setup before any deploy that depends on them:

| Item | Where in Setup |
|---|---|
| Platform Cache Partitions | Setup > Platform Cache > New Partition |
| Connected App OAuth secrets | Setup > App Manager > Edit > OAuth |
| Named Credential with OAuth token | Setup > Named Credentials > New > OAuth 2.0 |
| External Credential principals | Setup > Named Credentials > External Credentials > Principals |

A missing Platform Cache Partition fails at runtime, not deploy time. Document manual steps in your FEATURE_SPEC.md before the deploy starts.
