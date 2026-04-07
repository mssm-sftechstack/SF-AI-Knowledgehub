# SF CLI Cheatsheet

Quick reference for common Salesforce CLI commands.

---

## Authentication

### Login to Org

```bash
sf org login web --alias my-org
```

Opens browser for OAuth. Creates local alias `my-org`.

### List Authenticated Orgs

```bash
sf org list
```

Shows all logged-in orgs and their aliases.

### Set Default Org

```bash
sf config set target-org=my-org
```

All commands now target `my-org` by default.

### Logout

```bash
sf org logout --target-org=my-org
```

---

## Deployment

### Deploy Metadata

```bash
sf project deploy start --source-dir=force-app --target-org=my-org
```

Deploys all code in `force-app` directory.

### Deploy Specific Components

```bash
sf project deploy start --source-dir=force-app/main/default/classes/MyClass.cls --target-org=my-org
```

Deploy only one class.

### Validate Without Deploying

```bash
sf project deploy validate --source-dir=force-app --target-org=my-org
```

Checks for errors without making changes.

### Deploy with Specific Tests

```bash
sf project deploy start --source-dir=force-app --target-org=my-org --test-level=RunSpecifiedTests --tests=MyTest,AnotherTest
```

---

## Retrieval

### Retrieve Metadata to Local

```bash
sf project retrieve start --manifest=manifest/package.xml --target-org=my-org
```

Pulls code from org into local project.

### Retrieve Specific Type

```bash
sf project retrieve start --metadata=ApexClass --target-org=my-org
```

Retrieves all Apex classes.

---

## Scratch Orgs

### Create Scratch Org

```bash
sf org create scratch --definition-file=config/project-scratch-def.json --alias=feature-org
```

Creates temporary dev org using definition file.

### Delete Scratch Org

```bash
sf org delete scratch --target-org=feature-org
```

---

## Code Execution

### Run Anonymous Apex

```bash
sf apex run --file=scripts/test.apex --target-org=my-org
```

Executes Apex code from file.

### SOQL Query

```bash
sf data query --query="SELECT Id, Name FROM Account LIMIT 10" --target-org=my-org
```

Runs SOQL query and shows results.

---

## Source Tracking

### Check Source Status

```bash
sf project status
```

Shows changed files since last deploy.

### Push Local Changes

```bash
sf project deploy start --target-org=my-org
```

Deploys only changed files (for dev orgs).

### Pull Changes from Org

```bash
sf project retrieve start --target-org=my-org
```

Pulls recent changes from org.

---

## Common Errors & Fixes

### Error: "dubious ownership"

```
fatal: detected dubious ownership in repository at [path]
```

**Fix**: Allow repository ownership

```bash
git config --global --add safe.directory [path]
```

### Error: "permission denied"

```
Permission denied: sf command not found
```

**Fix**: Reinstall CLI globally

```bash
npm install --global @salesforce/cli
```

### Error: "INVALID_CROSS_REFERENCE_KEY"

```
Error message: INVALID_CROSS_REFERENCE_KEY
```

Usually means a referenced record doesn't exist. Check foreign keys in metadata.

### Error: "INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY"

User lacks FLS on a field. Check Permission Set.

### Error: "INVALID_DEPLOYMENT"

Metadata XML is malformed. Validate XML syntax.

---

## Useful Aliases

Add to `.bashrc` or `.zshrc`:

```bash
alias sfdeploy="sf project deploy start --source-dir=force-app --target-org=my-org"
alias sfretrieve="sf project retrieve start --manifest=manifest/package.xml --target-org=my-org"
alias sfdebug="sf apex run --file=scripts/debug.apex --target-org=my-org"
```

Then use:

```bash
sfdeploy
sfretrieve
sfdebug
```

---

## Flags Reference

| Flag | Purpose |
|------|---------|
| `--target-org` | Which org to target |
| `--alias` | Name for the org |
| `--async` | Run asynchronously (don't wait) |
| `--wait` | Wait X minutes for async to complete |
| `--test-level` | Which tests to run (RunLocalTests, RunSpecifiedTests) |
| `--tests` | Specific test classes to run |
| `--coverage` | Show code coverage % |
| `--json` | Output as JSON (for scripting) |

