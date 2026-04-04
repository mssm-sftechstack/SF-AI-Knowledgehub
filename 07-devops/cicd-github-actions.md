# CI/CD with GitHub Actions

**Automated validation on every PR. Automated deploy on merge. JWT auth so no passwords in your pipeline.**

---

## JWT Bearer Flow for CI Authentication

Passwords and session IDs expire. JWT auth uses a certificate you control. Here's the one-time setup:

**1. Generate a self-signed certificate:**
```bash
openssl genrsa -out server.key 2048
openssl req -new -x509 -key server.key -out server.crt -days 365 \
  -subj "/C=US/O=YourOrg/CN=salesforce-ci"
```

**2. Create a Connected App in Salesforce:**
- Enable OAuth
- Enable "Use Digital Signatures", upload `server.crt`
- Scopes: `api`, `refresh_token`, `offline_access`
- Enable "Require Secret for Web Server Flow": No
- Pre-authorize: the CI user must approve the app once, or use a pre-authorized policy

**3. Store secrets in GitHub:**
- `SF_JWT_KEY`: contents of `server.key` (the private key, never commit this)
- `SF_USERNAME`: the CI user's Salesforce username
- `SF_CLIENT_ID`: the Connected App's Consumer Key

---

## The Two Workflows Everyone Needs

### 1. PR Validation — validate on every PR to main

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

      - name: Authenticate to Org
        run: |
          echo "${{ secrets.SF_JWT_KEY }}" > server.key
          sf org login jwt \
            --username ${{ secrets.SF_USERNAME }} \
            --jwt-key-file server.key \
            --client-id ${{ secrets.SF_CLIENT_ID }} \
            --alias TargetOrg

      - name: Validate Deployment
        run: |
          sf project deploy validate \
            --source-dir force-app \
            --target-org TargetOrg \
            --test-level RunLocalTests \
            --wait 30

      - name: Clean up key file
        if: always()
        run: rm -f server.key
```

### 2. Deploy on Merge — full deploy when PR merges to main

```yaml
name: Deploy to Sandbox

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

      - name: Authenticate to Org
        run: |
          echo "${{ secrets.SF_JWT_KEY }}" > server.key
          sf org login jwt \
            --username ${{ secrets.SF_USERNAME }} \
            --jwt-key-file server.key \
            --client-id ${{ secrets.SF_CLIENT_ID }} \
            --alias TargetOrg

      - name: Deploy
        run: |
          sf project deploy start \
            --source-dir force-app \
            --target-org TargetOrg \
            --test-level RunLocalTests \
            --wait 30

      - name: Clean up key file
        if: always()
        run: rm -f server.key
```

---

## GitHub Secrets to Configure

| Secret | Value |
|---|---|
| `SF_JWT_KEY` | Full contents of `server.key` (multiline, include header and footer) |
| `SF_USERNAME` | The CI user's Salesforce username (e.g., `ci-user@yourorg.com`) |
| `SF_CLIENT_ID` | The Connected App's Consumer Key from Setup |

Add these at: Repository > Settings > Secrets and Variables > Actions > New Repository Secret.

---

## Branch Protection Rule

Require the validation workflow to pass before anyone can merge to main.

Repository > Settings > Branches > Add Branch Protection Rule:

- Branch name pattern: `main`
- Check: "Require status checks to pass before merging"
- Add status check: `validate` (the job name from the PR workflow)
- Check: "Require branches to be up to date before merging"

Without branch protection, the validation workflow runs but nothing enforces it. A developer can merge a PR with a failed validation check. The protection rule makes it mandatory.

---

## Common Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| `No authorization found` | JWT key or client ID mismatch | Verify the Connected App Consumer Key and that `server.crt` matches `server.key` |
| `user hasn't approved this connected app` | Connected App not pre-authorized | In Setup, edit the Connected App, set "Permitted Users" to "Admin approved users are pre-authorized", then assign the connected app to the CI user's profile or permission set |
| `INVALID_LOGIN` | Wrong username or sandbox URL | Add `--instance-url https://test.salesforce.com` for sandboxes |
| Tests fail on validate | Test class issue, not a CI issue | Run tests locally first with `sf project deploy validate --test-level RunLocalTests` |
| `--wait 30` times out | Large deploy, slow org | Increase wait time or use `--async` and poll the job ID |
