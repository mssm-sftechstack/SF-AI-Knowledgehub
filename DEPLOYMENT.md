# Deployment Strategy

How to deploy without breaking production.

---

## The 16-Phase Deployment Order

Deploy in this order. Each phase must be independently valid.

| # | Metadata Type | Why This Order |
|---|---|---|
| 1 | Custom Objects and Fields | Everything else references objects. They must exist first. |
| 2 | Custom Metadata Types | Used by Apex and Flows. Must exist before code. |
| 3 | Custom Settings | Code reads these. Deploy before code. |
| 4 | Custom Tabs | Permission Sets reference tabs. Tabs must exist first. |
| 5 | Permission Sets and Custom Permissions | Reference fields, tabs, and custom objects. Deploy after those. |
| 6 | Apex Classes (non-test) | Compiled against the org schema. Objects must exist. |
| 7 | Apex Test Classes | Depend on production Apex classes. |
| 8 | Triggers | Reference Apex handlers. Handlers must exist first. |
| 9 | Remote Site Settings and CSP Trusted Sites | Required before callouts. |
| 10 | Named Credentials and External Credentials | Depend on Remote Site Settings. |
| 11 | Flows | Reference objects, fields, Apex actions, Permission Sets. All must exist. |
| 12 | LWC and Aura Components | Reference Apex and objects. Apex must be deployed first. |
| 13 | FlexiPages | Reference LWC and Aura components. Components must exist. |
| 14 | Page Layouts and Compact Layouts | Reference fields. Fields must exist. |
| 15 | Applications | Reference tabs. Tabs must exist. |
| 16 | Profiles and Permission Set Assignments | Reference all other metadata. Come last. |

---

## Cross-Phase Gotchas

These errors stop deployments. Watch for them.

| Error | Cause | Fix |
|---|---|---|
| Permission Set `tabSettings` fails with "tab not found" | Deploying Permission Set tabSettings before the tab exists | Move tabSettings to same phase as the tab (phase 4-5) |
| Named Credential references undefined AuthProvider | AuthProvider not deployed yet | Deploy AuthProvider or Connected App first |
| LWC fails with "Apex method not found" | Deploying LWC before its Apex class | Deploy Apex classes before LWC |
| Custom App `defaultLandingTab` fails | Tab doesn't exist in org yet | Deploy tab in same phase as app or before |
| Trigger compilation fails with "handler class not found" | Deploying trigger before handler class | Deploy Apex handler first, trigger second |
| Flow activation fails with "Permission Set does not exist" | Permission Set referenced by flow isn't deployed | Deploy Permission Set before flow |
| Runtime error: "Platform Cache partition not found" | Platform Cache partitions can't be deployed via metadata | Create manually in Setup UI (not deployable) |

---

## Validate-Then-Deploy Pattern

Always validate first. It runs tests without actually deploying.

**Step 1: Validate (dry run)**

```bash
sf project deploy validate \
  --source-dir force-app \
  --target-org prod \
  --test-level RunLocalTests
```

Wait for the result. If validation passes, you get a job ID.

**Step 2: Deploy (if validation passed)**

Use the job ID from validation for a quick deploy that skips tests.

```bash
sf project deploy start \
  --source-dir force-app \
  --target-org prod \
  --job-id [job-id-from-validation] \
  --wait 30
```

Why two steps. Validation runs all tests (slow). Once it passes, deploy is instant.

---

## GitHub Actions CI/CD

Automated validation on PR, automated deploy on merge.

### Prerequisites

Set up JWT authentication for CI (one time).

**1. Generate a certificate:**

```bash
openssl genrsa -out server.key 2048
openssl req -new -x509 -key server.key -out server.crt -days 365 \
  -subj "/C=US/O=YourOrg/CN=salesforce-ci"
```

**2. Create a Connected App in Salesforce:**
- Setup → Apps → App Manager → New
- Name: "CI/CD Pipeline"
- Enable OAuth
- Enable "Use Digital Signatures"
- Upload your `server.crt` file
- Scopes: api, refresh_token, offline_access
- Save. Copy the Consumer Key (Client ID).

**3. Store secrets in GitHub:**
- Go to your repo → Settings → Secrets and variables → Actions
- Add:
  - `SF_JWT_KEY`: Contents of server.key (private key)
  - `SF_USERNAME`: CI user's Salesforce username
  - `SF_CLIENT_ID`: Connected App's Consumer Key
  - `SF_INSTANCE_URL`: Your Salesforce instance URL (e.g., https://myorg.my.salesforce.com)

### Workflow 1: PR Validation

Create `.github/workflows/validate-pr.yml`:

```yaml
name: Validate PR

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SF CLI
        run: npm install -g @salesforce/cli

      - name: Authenticate with JWT
        run: |
          sf org login jwt \
            --client-id ${{ secrets.SF_CLIENT_ID }} \
            --jwt-key-file <(echo "${{ secrets.SF_JWT_KEY }}") \
            --username ${{ secrets.SF_USERNAME }} \
            --instance-url ${{ secrets.SF_INSTANCE_URL }} \
            --set-default

      - name: Validate PR changes
        run: |
          sf project deploy validate \
            --source-dir force-app \
            --test-level RunLocalTests \
            --wait 30
```

**What it does:**
- Runs on every PR to main
- Authenticates using JWT
- Runs validation (all tests, no deploy)
- Fails the PR if validation fails

### Workflow 2: Deploy on Merge

Create `.github/workflows/deploy-main.yml`:

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SF CLI
        run: npm install -g @salesforce/cli

      - name: Authenticate with JWT
        run: |
          sf org login jwt \
            --client-id ${{ secrets.SF_CLIENT_ID }} \
            --jwt-key-file <(echo "${{ secrets.SF_JWT_KEY }}") \
            --username ${{ secrets.SF_USERNAME }} \
            --instance-url ${{ secrets.SF_INSTANCE_URL }} \
            --set-default

      - name: Validate before deploy
        run: |
          VALIDATION_ID=$(sf project deploy validate \
            --source-dir force-app \
            --test-level RunLocalTests \
            --wait 30 \
            --json | jq -r .result.id)
          echo "VALIDATION_ID=$VALIDATION_ID" >> $GITHUB_ENV

      - name: Deploy with validation result
        run: |
          sf project deploy start \
            --source-dir force-app \
            --job-id ${{ env.VALIDATION_ID }} \
            --wait 30

      - name: Report result
        if: always()
        run: echo "Deploy completed. Check Salesforce Setup → Deployment Status."
```

**What it does:**
- Runs on every merge to main
- Validates first
- Deploys using the validation job ID (fast)
- Skips test re-run (already validated)

---

## Manual Deployment Checklist

If deploying without CI/CD:

- [ ] Create a feature branch (git checkout -b feature/my-feature)
- [ ] Make your changes in force-app/
- [ ] Commit to git
- [ ] Create a PR, get code review approval
- [ ] Merge to main
- [ ] Pull latest main locally
- [ ] Run: `sf project deploy validate --source-dir force-app --target-org prod --test-level RunLocalTests`
- [ ] Wait for validation to pass (10-30 minutes)
- [ ] Run: `sf project deploy start --source-dir force-app --target-org prod --job-id [validation-job-id]`
- [ ] Monitor in Salesforce Setup → Deployment Status

---

## What NOT to Deploy

- Test data or fixtures
- Secrets or credentials (API keys, passwords)
- Local config files (.env, local.json)
- node_modules or dependencies (use package-lock.json instead)
- Large data files

Use `.gitignore` to exclude these.

---

## Rollback

If a deployment breaks production, here's the fix.

**Option 1: Revert the commit**

```bash
git revert HEAD
git push
# This creates a new commit that undoes the bad one
```

**Option 2: Manual rollback (for urgent fixes)**

Delete the problematic metadata from the org via Setup UI. Document why.

**Option 3: Database rollback (last resort)**

If data was corrupted, Salesforce support can restore from backup (within 5 days).

---

## Monitoring Deployments

Watch what deployed:

```bash
# Show deployment status
sf project deploy report --job-id [deployment-job-id]

# Watch realtime
sf project deploy start ... --wait 60 # waits up to 60 minutes
```

In Salesforce:

Setup → Deployment Status → click the deployment to see details, errors, and test results.
