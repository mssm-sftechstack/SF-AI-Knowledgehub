---
name: sf-deploy
description: >
  Salesforce deployment skill. Covers metadata deployment, retrieval, validation,
  scratch org management, validate-then-deploy pattern, and CI/CD workflows. Always
  asks user for confirmation before any deployment command. Based on Salesforce CLI
  Reference and DX Developer Guide (API 62.0).
allowed-tools: Bash(sf project deploy* --target-org <your-org-alias>*), Bash(sf project retrieve*), Bash(sf apex run test* --target-org <your-org-alias>*), Bash(sf org list*), Bash(sf org create scratch*), Bash(sf org delete scratch*), Bash(git*)
---

# Salesforce Deployment Skill

**Metadata deployment, retrieval, validation, scratch org management, validate-then-deploy pattern, and CI/CD authentication.**

## Rule 1 -- Always Ask Before Deploying

Before running ANY deployment command, stop and confirm:
```
"I'm about to deploy to: [org alias]
Metadata: [exact list]
Command: [exact command]
Shall I proceed? (yes/no)"
```

Do NOT proceed until the user explicitly says yes.

---

## Rule 2 -- Confirm Target Org

Never deploy to production without explicit written approval. If asked to deploy to an unrecognised org, stop and confirm first.

---

## Standard Deployment Workflow

### Step 1 -- Confirm with user

### Step 2 -- Run security review

### Step 3 -- Validate (dry run) first

```bash
sf project deploy validate \
  --source-dir force-app \
  --target-org <your-org-alias> \
  --test-level RunLocalTests \
  --wait 30
```

### Step 4 -- Run tests with coverage

```bash
sf apex run test \
  --test-level RunLocalTests \
  --code-coverage \
  --target-org <your-org-alias> \
  --wait 60
```

### Step 5 -- Quick deploy after successful validation

```bash
sf project deploy quick \
  --job-id <validationJobId> \
  --target-org <your-org-alias> \
  --wait 10
```

### Step 6 -- Stage specific files (never `git add .`)

```bash
git add force-app/main/default/classes/WeatherService.cls
git add force-app/main/default/classes/WeatherService.cls-meta.xml
git commit -m "feat(weather): add WeatherService callout"
```

---

## Common Deployment Commands

```bash
sf project deploy start --source-dir force-app --target-org <your-org-alias>
sf project deploy start --metadata ApexClass --target-org <your-org-alias>
sf project deploy start --metadata ApexClass,ApexTrigger --target-org <your-org-alias>
sf project deploy start --metadata LightningComponentBundle:weatherDashboard --target-org <your-org-alias>
sf project deploy start --manifest manifest/package.xml --target-org <your-org-alias>
sf project deploy report --target-org <your-org-alias>
sf project deploy cancel --job-id <jobId> --target-org <your-org-alias>
sf project retrieve start --metadata ApexClass:WeatherService --target-org <your-org-alias>
sf project generate manifest --output-dir ./manifest --from-org <your-org-alias>
```

---

## Scratch Org -- Local Dev and Testing

```bash
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias FeatureOrg --duration-days 30 --set-default

sf project deploy start --target-org FeatureOrg
sf project retrieve start --target-org FeatureOrg
sf org open --target-org FeatureOrg
sf org list
sf org delete scratch --target-org FeatureOrg --no-prompt
```

Scratch orgs are for development only -- never use as a production deploy target.

---

## CI/CD Authentication -- JWT Bearer Flow

```bash
sf org login jwt \
  --client-id "$SALESFORCE_CLIENT_ID" \
  --jwt-key-file ./server.key \
  --username "$SALESFORCE_USERNAME" \
  --alias ci-org --set-default

rm ./server.key
```

**Reference:** [JWT Bearer Flow](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm)

---

## Destructive Changes (Extra Confirmation Required)

Before any destructive deploy show the user the full `destructiveChanges.xml` and wait for explicit yes.

```bash
sf project deploy start \
  --manifest manifest/package.xml \
  --post-destructive-changes destructive/destructiveChanges.xml \
  --target-org <your-org-alias> \
  --wait 30
```

---

## Deployment Order for Complex Changes

1. Custom Objects and Fields
2. Custom Metadata Types
3. Custom Settings
4. **Custom Tabs** -- must exist BEFORE any Permission Set references them
5. Permission Sets and Custom Permissions
6. Apex Classes (non-test)
7. Apex Test Classes
8. Triggers
9. Remote Site Settings
10. Named Credentials and External Credentials
11. Flows
12. LWC / Aura Components
13. FlexiPages
14. Page Layouts and Compact Layouts
15. Applications
16. Profiles and Permission Set Assignments

---

## Wave Deployment -- Resolving Dependency Deadlocks

```
Wave 1: deploy the dependency first
sf project deploy start \
  --metadata CustomTab:City_Weather__c \
  --target-org <alias> \
  --wait 10

Wave 2: deploy everything else
sf project deploy start \
  --source-dir force-app \
  --target-org <alias> \
  --test-level RunLocalTests \
  --wait 30
```

Common dependency pairs:

| Component | Depends On | Deploy First |
|---|---|---|
| PermissionSet (tabSettings) | CustomTab | Tab in Wave 1 |
| PermissionSet (objectPermissions) | CustomObject | Object in Wave 1 |
| FlexiPage | LWC component | LWC in Wave 1 |
| Flow | Custom Object / Field | Schema in Wave 1 |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `MISSING_TYPE` | Missing dependency | Add to manifest, deploy in correct order |
| `Test coverage is 0%` | Test class not deployed | Deploy test class first |
| `Dependent class is invalid` | References deleted class | Fix all references first |
| `Flow version already exists` | Same API name conflict | Retrieve and deactivate old version |
| `Named credential not found` | Named Credential not deployed | Deploy NamedCredential + ExternalCredential first |
| `tabVisibilities invalid at this location in type PermissionSet` | Wrong element name -- only valid on Profiles | Use `<tabSettings>` in `.permissionset-meta.xml` |

---

## Deployment Checklist

- [ ] User confirmed target org and metadata list
- [ ] Security review completed
- [ ] `sf project deploy validate` passed with `RunLocalTests`
- [ ] 90%+ Apex code coverage confirmed
- [ ] Deployment order followed for multi-metadata changes
- [ ] Destructive changes reviewed and confirmed by user
- [ ] Quick deploy used after successful validation
- [ ] SFDX Auth URL NOT used for CI (JWT Bearer Flow only)

---

## Sources

- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
- [Salesforce CLI Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_top.htm)
- [JWT Bearer Flow](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm)
- [Scratch Orgs](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs.htm)
