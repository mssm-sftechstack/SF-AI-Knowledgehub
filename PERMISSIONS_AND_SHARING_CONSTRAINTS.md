# Enterprise Permissions & Sharing Constraints

**CRITICAL DIRECTIVE FOR AI:** Salesforce is actively deprecating permissions on Profiles. You must adopt a modern, composable security model based entirely on Permission Sets and Permission Set Groups. 

## 1. The Profile Ban
If a user asks you how to grant access to an Object, Field, Apex Class, or Tab, you must **NEVER** suggest modifying a Profile.
* **The Rule:** All CRUD, FLS, and App-level access must be assigned via Permission Sets.
* Profiles should ONLY be used for defaults: Page Layout assignments, Login IP Ranges, Default Record Types, and Session Settings.

## 2. Explicit Class Sharing (`with sharing`)
AI models default to writing `public class MyClass { ... }`. In Salesforce, omitting the sharing declaration defaults the class to the sharing context of its caller, which is a massive security vulnerability.

* **The Rule:** Every single Apex class must explicitly declare its sharing model. 
* Default to `with sharing`.
* If you must use `without sharing` to bypass the role hierarchy for a system-level update, you must add a comment explicitly explaining the architectural justification.

```apex
// ✅ MANDATORY CLASS DEFINITION
public with sharing class SecureDataService {
    // Logic runs in User Context
}
```

## 3. The Custom Permission Toggle
AI models love to hardcode Profile IDs or Role names to bypass logic (e.g., `if (userInfo.getProfileId() == '00e...Label')`). This will fail deployment to production.

* **The Rule:** NEVER hardcode IDs or use Profile names for logic bypasses. 
* ALWAYS use Custom Permissions. Instruct the user to create a Custom Permission (e.g., `Bypass_Account_Validation`) and check for it using `FeatureManagement.checkPermission()`.

```apex
// ✅ Correct Logic Bypass Pattern
if (FeatureManagement.checkPermission('Bypass_Account_Validation')) {
    return; // Bypass logic for integration user
}
```