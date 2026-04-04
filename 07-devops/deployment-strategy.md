# Deployment Strategy

**A deployment that fails halfway through a production deploy at 2am is the worst Salesforce experience. Phase your deploys so each one is independently valid.**

---

## The Phase Principle

Each phase must be deployable on its own without depending on a later phase. If Phase 1 fails, Phase 2 never runs. If Phase 2 depends on Phase 1 and Phase 1 fails, you're stuck.

Design each phase so it can be deployed, verified in production, and rolled back independently.

---

## The Definitive Deployment Order

| # | Metadata Type | Why This Order |
|---|---|---|
| 1 | Custom Objects and Fields | Everything else references objects. They must exist first. |
| 2 | Custom Metadata Types | Used by Apex classes and Flows. Must exist before code that reads them. |
| 3 | Custom Settings | Same as Custom Metadata: code reads these. Deploy before the code. |
| 4 | Custom Tabs | Permission Sets reference tabs. Tabs must exist before the PS that references them. |
| 5 | Permission Sets and Custom Permissions | Reference fields and tabs. Deploy after objects, fields, and tabs. |
| 6 | Apex Classes (non-test) | Compiled against the org schema. Objects and fields must exist first. |
| 7 | Apex Test Classes | Depend on production Apex classes being present. |
| 8 | Triggers | Reference Apex handler classes. Handlers must exist first. |
| 9 | Remote Site Settings and CSP Trusted Sites | Required before any component makes external callouts. |
| 10 | Named Credentials and External Credentials | Depend on Remote Site Settings and AuthProviders. |
| 11 | Flows | Reference objects, fields, Apex actions, and Permission Sets. All must exist. |
| 12 | LWC and Aura Components | Reference Apex classes and custom objects. Apex must be deployed first. |
| 13 | FlexiPages | Reference LWC components. Components must exist first. |
| 14 | Page Layouts and Compact Layouts | Reference fields and objects. Objects must exist first. |
| 15 | Applications | Reference tabs. Tabs must exist first. |
| 16 | Profiles and Permission Set Assignments | Reference all other metadata. Must come last. |

---

## Cross-Phase Traps That Cause Failures

| Trap | Symptom | Fix |
|---|---|---|
| PermissionSet `tabSettings` before the Tab is deployed | Invalid reference error: tab not found | Move `tabSettings` to the same deploy phase as the Tab (step 4-5) |
| Named Credential before AuthProvider | Dependency error: auth provider not found | Deploy AuthProvider first, then Named Credential |
| LWC component before the Apex class it imports | Compile error: Apex method not found | Always deploy Apex before LWC |
| Custom App `defaultLandingTab` references an undeployed Tab | Error: tab does not exist | Deploy Tab in the same phase as the App or before it |
| Trigger deployed before its handler class | Compile error: class not found | Apex handler first, trigger second |
| Flow referencing a Permission Set that isn't deployed yet | Validation error during flow activation | Deploy PS before Flow |
| Platform Cache Partition referenced in code before it exists in Setup | Runtime error: partition not found | Create manually in Setup UI first. It's not deployable via metadata. |

---

## The Validate-Then-Deploy Pattern

For production deploys, always validate first. Validation runs all tests without actually deploying. If validation passes, you get a job ID you can use for a quick deploy that skips tests.

```bash
# Step 1: validate (dry run — runs all tests, nothing actually deployed)
sf project deploy validate \
  --source-dir force-app \
  --target-org prod \
  --test-level RunLocalTests \
  --wait 30

# Step 1 output includes a job ID, e.g.: 0Af4x000003yDFGCAM

# Step 2: quick deploy (uses validated components, skips test re-run)
sf project deploy quick \
  --job-id 0Af4x000003yDFGCAM \
  --target-org prod
```

Quick deploy is valid for 10 days after validation. Use it during your deploy window so you don't re-run tests in production at 2am.

---

## Manual Pre-Deploy Steps

Some things can't be deployed via metadata. Document them in your FEATURE_SPEC.md before any deploy.

| Item | Why Manual | How |
|---|---|---|
| Platform Cache Partitions | Not supported in metadata API | Setup > Platform Cache > New Partition |
| Connected App OAuth setup | Client secret not deployable | Setup > App Manager > Edit > OAuth |
| Named Credential with OAuth flow | OAuth token requires browser redirect | Setup > Named Credentials > New > OAuth 2.0 |
| External Credential principal | User credentials not deployable | Setup > Named Credentials > External Credentials > Principals |

If a colleague is doing the deploy without you, write these steps down before they start. A missing Platform Cache Partition fails at runtime, not at deploy time, so it's easy to miss.
