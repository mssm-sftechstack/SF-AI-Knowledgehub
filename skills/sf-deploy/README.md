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

Confirm 90%+ coverage. Do not proceed if any test fails.

### Step 5 -- Quick deploy after successful validation

```bash
sf project deploy quick \
  --job-id <validationJobId> \
  --target-org <your-org-alias> \
  --wait 10
```

### Step 6 -- Stage specific files (never `git add .` -- risks staging secrets)

```bash
git add force-app/main/default/classes/WeatherService.cls
git add force-app/main/default/classes/WeatherService.cls-meta.xml
git commit -m "feat(weather): add WeatherService callout"
```

---

## Common Deployment Commands

```bash
# deploy all source
sf project deploy start --source-dir force-app --target-org <your-org-alias>

# deploy specific metadata types
sf project deploy start --metadata ApexClass --target-org <your-org-alias>
sf project deploy start --metadata ApexClass,ApexTrigger --target-org <your-org-alias>
sf project deploy start --metadata LightningComponentBundle:weatherDashboard --target-org <your-org-alias>
sf project deploy start --metadata Flow:Send_Alert_Opportunity_After_Save --target-org <your-org-alias>

# deploy from manifest
sf project deploy start --manifest manifest/package.xml --target-org <your-org-alias>

# check deployment status
sf project deploy report --target-org <your-org-alias>

# cancel in-progress deployment
sf project deploy cancel --job-id <jobId> --target-org <your-org-alias>

# retrieve metadata
sf project retrieve start --metadata ApexClass:WeatherService --target-org <your-org-alias>
sf project retrieve start --manifest manifest/package.xml --target-org <your-org-alias>

# generate manifest from org
sf project generate manifest --output-dir ./manifest --from-org <your-org-alias>
```

---

## Scratch Org -- Local Dev and Testing

```bash
# create
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias FeatureOrg --duration-days 30 --set-default

# push source to scratch org
sf project deploy start --target-org FeatureOrg

# pull changes from scratch org back to source
sf project retrieve start --target-org FeatureOrg

# open scratch org
sf org open --target-org FeatureOrg

# list all orgs
sf org list

# delete when done
sf org delete scratch --target-org FeatureOrg --no-prompt
```

Scratch orgs are for development only -- never use as a production deploy target.

---

## CI/CD Authentication -- JWT Bearer Flow

For CI/CD pipelines, use JWT Bearer Flow -- not SFDX Auth URL. JWT is more secure because it uses a private key that is never exposed.

```bash
sf org login jwt \
  --client-id "$SALESFORCE_CLIENT_ID" \
  --jwt-key-file ./server.key \
  --username "$SALESFORCE_USERNAME" \
  --alias ci-org --set-default

# remove the key file immediately after use
rm ./server.key
```

**Reference:** [JWT Bearer Flow](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm)

---

## Destructive Changes (Extra Confirmation Required)

Before any destructive deploy:
1. Show full `destructiveChanges.xml` to the user
2. Ask: "This permanently deletes the listed metadata from `<your-org-alias>`. Confirm? (yes/no)"
3. Wait for explicit confirmation

```bash
sf project deploy start \
  --manifest manifest/package.xml \
  --post-destructive-changes destructive/destructiveChanges.xml \
  --target-org <your-org-alias> \
  --wait 30
```

---

## Deployment Order for Complex Changes

Follow this order. Items referenced by later items must land in the org first.

1. Custom Objects and Fields
2. Custom Metadata Types
3. Custom Settings
4. **Custom Tabs** -- must exist in org BEFORE any Permission Set references them
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

When component A references component B and B doesn't exist in the org yet, a single-batch deploy fails even though both are in the same changeset. The Metadata API validates each component against the current org state, not against other components in the same batch.

**Fix -- split into waves:**

```
Wave 1: deploy the dependency first (e.g. the Tab)
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
| Application (tabs) | CustomTab | Tab in Wave 1 |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `MISSING_TYPE` | Missing dependency | Add to manifest, deploy in correct order |
| `Test coverage is 0%` | Test class not deployed | Deploy test class first |
| `Dependent class is invalid` | References deleted class | Fix all references before deploying |
| `Flow version already exists` | Same API name conflict | Retrieve and deactivate old version |
| `Named credential not found` | Named Credential not deployed | Deploy NamedCredential + ExternalCredential first |
| `Cannot set endpoint` | Remote Site Setting missing | Add domain to Remote Site Settings |
| `tabVisibilities invalid at this location in type PermissionSet` | Wrong element name -- `tabVisibilities` is for Profiles only | Use `<tabSettings>` in `.permissionset-meta.xml`. Group all same-type elements together in XSD order |

---

## Deployment Checklist

- [ ] User confirmed target org and metadata list before running any command
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
- [Well-Architected: Adaptable](https://architect.salesforce.com/well-architected/adaptable/evolvable)
