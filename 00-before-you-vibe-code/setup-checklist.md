# Pre-Flight Checklist -- Before You Vibe Code

**20 things to verify before writing your first AI-assisted Salesforce prompt.**

---

Work through this list top to bottom before starting a new feature or a new AI-assisted session. Each item takes less than two minutes to verify. Skipping items in the middle is where most failed deployments and security review rejections originate.

---

## Checklist

| # | Item | Why it matters |
|---|---|---|
| 1 | Context file loaded for your AI tool (`CLAUDE.md` / `.cursorrules` / `copilot-instructions.md`) | Without this, the AI generates code with no knowledge of governor limits, security rules, or deployment order. Every other item on this list should be reflected in that file. |
| 2 | SF CLI installed and authenticated (`sf org display --target-org <alias>` returns org details) | All deployments, retrievals, and test runs go through the CLI. If it's not authenticated, nothing deploys. |
| 3 | VS Code with Salesforce Extension Pack installed | The extension provides Apex IntelliSense, LWC support, and inline error detection. Working without it means missing obvious errors before they reach deployment. |
| 4 | Working in a Scratch Org or Developer Org -- not production | Production orgs have real users and real data. Changes made directly in production bypass version control and can't be rolled back cleanly. |
| 5 | `sfdx-project.json` exists with `sourceApiVersion` set to `62.0` | The API version controls which metadata features are available. Mismatched versions cause silent deploy failures or missing features. |
| 6 | You know which object you're working on and whether a trigger already exists | Salesforce allows only one trigger per object per event. Creating a second trigger causes unpredictable execution order and logic conflicts. |
| 7 | You've checked for existing Apex classes before creating new ones | Duplicate service classes with similar names split logic and make testing unreliable. Always extend an existing class before creating a new one. |
| 8 | You understand the OWD (Org-Wide Default) for the objects you'll query | If an object's OWD is Private, a `with sharing` class will silently filter out records the user can't see. Queries may return fewer rows than expected. |
| 9 | You know which Permission Set the feature will use | Every new field, object, and tab needs field permissions in a Permission Set. Knowing this upfront means the AI generates the correct metadata alongside the code. |
| 10 | You've read any restricted picklist field's `.field-meta.xml` before writing code that sets it | Restricted picklists reject any value not in the field's metadata. The AI will guess plausible-sounding values that don't exist in your org. Always check the exact `<fullName>` strings. |
| 11 | You have a non-production org for this work (Scratch Org or Sandbox) | Make sure the org actually exists and is accessible before starting. A deleted scratch org mid-session wastes time. |
| 12 | You know the deployment order for your feature | Objects must deploy before fields, tabs before Permission Sets that reference them, Apex classes before triggers. Getting this wrong produces cryptic metadata deploy errors. See the [deployment order reference](../03-deployment/deployment-order.md). |
| 13 | No hardcoded IDs anywhere in your prompts to the AI | If you paste a Record Type ID or org ID into a prompt, the AI will use it in the generated code. That ID is meaningless in every org except the one you copied it from. |
| 14 | You'll ask the AI to write a test class alongside every new Apex class | Test classes aren't optional. Salesforce requires 75% overall coverage to deploy to production. 90%+ per class is the practical standard. Asking for both at once is faster than adding tests later. |
| 15 | You'll ask the AI to review its own code against multi-tenant rules before finalising | A self-review prompt ("check this for SOQL in loops, missing `with sharing`, and hardcoded IDs") catches most common mistakes before you deploy. |
| 16 | Your AI tool's context file specifies API version `62.0` | Without a specified API version, the AI may generate syntax for an older API that's deprecated or behaves differently. |
| 17 | You understand the difference between before-save and after-save flows | Before-save flows run before the record is committed and can't make DML calls. After-save flows run after commit and can query related records. Choosing wrong means the flow can't do what you need. |
| 18 | You know whether your feature needs `with sharing` or `inherited sharing` | `with sharing` enforces the running user's record visibility. `inherited sharing` defers to the caller's context. Service classes called from both trusted and untrusted contexts usually need `inherited sharing`. |
| 19 | Named Credentials exist for any external API calls your feature makes | Hardcoded endpoint URLs break between environments and fail security review. Named Credentials store the URL and authentication in org configuration, separate from code. |
| 20 | You have a phased deployment plan before writing any code | Complex features that span objects, classes, and Permission Sets need to deploy in a specific order. Writing the plan before writing code means you discover ordering conflicts before they cost you time. |

---

## Quick verification commands

Run these to confirm your environment is ready:

```bash
# Confirm CLI is installed and current
sf --version

# Confirm you are authenticated to the right org
sf org display --target-org <your-alias>

# Confirm your project structure is valid
sf project validate --source-dir force-app

# Confirm no existing triggers on your target object
sf data query \
  --query "SELECT Name FROM ApexTrigger WHERE TableEnumOrId = 'Account'" \
  --target-org <your-alias>
```

---

## What to do if an item is not ready

| Item not ready | Action before proceeding |
|---|---|
| No context file | Copy the template from [02-ai-tool-setup/](../02-ai-tool-setup/) and customise for your org |
| CLI not authenticated | Follow [01-environment-setup/sf-cli-setup.md](../01-environment-setup/sf-cli-setup.md) |
| No scratch org | Follow [01-environment-setup/scratch-org-quickstart.md](../01-environment-setup/scratch-org-quickstart.md) |
| Trigger already exists | Read the existing trigger handler class, then prompt the AI to add a method to it |
| No Named Credential | Create it in Setup before asking the AI to write the callout class |
| Restricted picklist not read | Run `sf project retrieve start` for that field's metadata, then read the `.field-meta.xml` file |
