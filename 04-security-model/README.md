# Security Model

**Salesforce's security model has 5 layers. Most bugs come from misunderstanding which layer does what.**

| Layer | Controls | Enforced by |
|---|---|---|
| Object permissions | Can the user see this object at all? | Profile + Permission Set |
| Field-level security (FLS) | Can the user see or edit this field? | Profile + Permission Set |
| Record visibility (OWD) | Which records can the user see by default? | Org-Wide Defaults |
| Record sharing | Share specific records beyond OWD | Role hierarchy, sharing rules, manual sharing |
| Apex sharing | Code-level sharing enforcement | `with sharing`, `without sharing`, `inherited sharing` |

The layers stack. A user needs object access before field access matters. They need record visibility before field values on that record are reachable. Apex sharing controls which records are returned in queries, not which fields.

## Contents

| Page | What it covers |
|---|---|
| [Permission Sets Over Profiles](permission-sets-over-profiles.md) | Why PS is the right approach, what goes in them, the fieldPermissions trap |
| [FLS and CRUD Enforcement](fls-crud-enforcement.md) | WITH SECURITY_ENFORCED, stripInaccessible(), USER_MODE, CRUD checks |
| [Sharing Model](sharing-model.md) | OWD, role hierarchy, sharing rules, with/without/inherited sharing |
