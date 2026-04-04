# Salesforce Security Skill

**Security review checklist. Run before every deployment. Covers SOQL injection, XSS, FLS, CRUD, sharing gaps, hardcoded credentials, insecure callouts, and more.**

## Run This Before Every Deployment

Review all changed files. Report every finding with:
- **Severity**: CRITICAL / HIGH / MEDIUM / LOW
- **File and line** reference
- **Exact fix** required

---

## 1. SOQL Injection (CRITICAL)

Dynamic SOQL with user-controlled input is the #1 Salesforce vulnerability.

```apex
// VULNERABLE
String q = 'SELECT Id FROM Account WHERE Name = \'' + userInput + '\'';
Database.query(q);

// SAFE - bind variables always
String soql = 'SELECT Id FROM Account WHERE Name = :userInput';
Database.query(soql);

// SAFE - whitelist for dynamic field names
Set<String> ALLOWED_FIELDS = new Set<String>{'Name', 'Phone', 'BillingCity'};
if (!ALLOWED_FIELDS.contains(fieldName)) {
    throw new SecurityException('Invalid field: ' + fieldName);
}
```

Flag: string concatenation inside `Database.query()`, `Search.query()`, `Database.countQuery()`.

**Reference:** [Apex SOQL Injection Prevention](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/pages_security_tips_soql_injection.htm)

---

## 2. FLS Violations (HIGH)

Field-Level Security must be enforced on all SOQL. Use one of two approaches:

```apex
// approach 1: WITH SECURITY_ENFORCED at query time (throws if field inaccessible)
List<Account> accs = [SELECT Id, Revenue__c FROM Account WITH SECURITY_ENFORCED];

// approach 2: stripInaccessible (strips fields instead of throwing)
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Revenue__c FROM Account]
);
List<Account> accs = (List<Account>) decision.getRecords();
```

Flag: SOQL without `WITH SECURITY_ENFORCED` or `stripInaccessible()`.
Exception: Custom Metadata (`__mdt`) and Custom Settings are exempt.

**Reference:** [Enforcing FLS](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_with_security_enforced.htm)

---

## 3. CRUD Violations (HIGH)

Check object-level permissions before every DML operation.

```apex
if (!Schema.sObjectType.Account.isCreateable()) {
    throw new AuraHandledException('No permission to create Accounts.');
}
insert accounts;

if (!Schema.sObjectType.Account.isUpdateable()) {
    throw new AuraHandledException('No permission to update Accounts.');
}
update accounts;

if (!Schema.sObjectType.Account.isDeletable()) {
    throw new AuraHandledException('No permission to delete Accounts.');
}
delete accounts;
```

**Reference:** [Enforcing Object and Field Permissions](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_security_fls_enforced.htm)

---

## 4. Sharing Model (HIGH)

```apex
// VIOLATION - no sharing declaration (inherits unpredictable context)
public class AccountService { }

// VIOLATION - without sharing with no justification
public without sharing class AccountService { }

// CORRECT - always declare with sharing
public with sharing class AccountService { }

// CORRECT - inherited sharing for utility classes called from multiple contexts
public inherited sharing class AccountSelector { }

// CORRECT - documented exception for system-level operations
public with sharing class AccountService {
    // runs elevated for scheduled integration - approved by architect
    private without sharing class SystemOps {
        static void runElevated() { /* ... */ }
    }
}
```

`inherited sharing` is the right choice for selector/utility classes called from both user-context code and system-context code.

**Reference:** [Using with sharing, without sharing, and inherited sharing](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_keywords_sharing.htm)

---

## 5. XSS in LWC / Aura (HIGH)

LWC template bindings `{expression}` are auto-escaped. Always prefer them.

```javascript
// VULNERABLE
this.template.querySelector('.content').innerHTML = userHtml;
eval(responseData.script);
new Function(userCode)();

// SAFE
this.template.querySelector('.content').textContent = userInput;
```

Flag: `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `eval()`, `Function()` with external data.

**Reference:** [LWC Security Best Practices](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_security_xss.htm)

---

## 6. Hardcoded Credentials / IDs (HIGH)

```apex
// VIOLATIONS
String profileId = '00e5g000001AbCdEAF'; // hardcoded ID
String apiKey    = 'sk-live-abc123secret'; // hardcoded credential

// CORRECT - Custom Metadata for config
String apiKey = [SELECT API_Key__c FROM My_Config__mdt WHERE DeveloperName = 'Default' LIMIT 1].API_Key__c;

// CORRECT - Named Credential for callout secrets
req.setEndpoint('callout:MyNamedCredential/endpoint');

// CORRECT - dynamic record type lookup (never hardcode record type IDs)
Id rtId = Schema.SObjectType.Opportunity
    .getRecordTypeInfosByDeveloperName()
    .get('Enterprise')
    .getRecordTypeId();
```

Flag: 15/18-char Salesforce IDs as string literals; variables named `password`, `secret`, `apiKey`, `token` with hardcoded string values.

**Reference:** [Secure Coding Hardcoding Credentials](https://developer.salesforce.com/docs/atlas.en-us.secure_coding_guide.meta/secure_coding_guide/secure_coding_hardcoding_credentials.htm)

---

## 7. Insecure Callouts (HIGH)

```apex
// VIOLATION - HTTP not HTTPS
req.setEndpoint('http://api.example.com/data');

// CORRECT - Named Credential always
req.setEndpoint('callout:OpenWeatherMap_NC/data/2.5/weather?q=' + city);
req.setTimeout(10000); // always set a timeout
```

Flag: HTTP endpoints, missing timeouts, credentials in endpoint strings.

**Reference:** [Named Credentials](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_callouts_named_credentials.htm)

---

## 8. Insecure JSON Deserialization (HIGH)

```apex
// VULNERABLE - type from user input
Type t = Type.forName(userProvidedTypeName);
Object obj = JSON.deserialize(body, t);

// SAFE - always use a concrete, known type
MyWrapper data = (MyWrapper) JSON.deserialize(body, MyWrapper.class);
```

Flag: `Type.forName()` with user-controlled input.

**Reference:** [Apex JSON and Deserialization Security](https://developer.salesforce.com/docs/atlas.en-us.secure_coding_guide.meta/secure_coding_guide/secure_coding_apex.htm)

---

## 9. SOQL Missing LIMIT (HIGH)

```apex
// VIOLATION - no limit, will fail at 50,001 rows
List<Account> accs = [SELECT Id FROM Account WHERE IsActive__c = true];

// CORRECT - always add LIMIT, or use QueryLocator for large sets
List<Account> accs = [SELECT Id FROM Account WHERE IsActive__c = true LIMIT 1000];
```

Flag: SOQL on standard or custom objects without `LIMIT`.

---

## 10. Open Redirect (MEDIUM)

```javascript
// VULNERABLE - unvalidated redirect
window.location.href = this.userProvidedUrl;

// SAFE - validate URL before navigating
safeNavigate(url) {
    try {
        const parsed = new URL(url);
        const allowed = ['yourdomain.com', 'salesforce.com', 'force.com'];
        if (parsed.protocol !== 'https:') return;
        if (!allowed.some(d => parsed.hostname.endsWith(d))) return;
        this[NavigationMixin.Navigate]({ type: 'standard__webPage', attributes: { url } });
    } catch (e) {
        // invalid URL - do not navigate
    }
}
```

---

## 11. Debug Log Exposure (MEDIUM)

```apex
// VIOLATIONS - PII and secrets in logs
System.debug('Customer SSN: ' + customer.SSN__c);
System.debug('API token: ' + token);

// CORRECT - log IDs and non-sensitive context only
System.debug(LoggingLevel.FINE, 'Processing record: ' + record.Id);
```

---

## 12. Guest / Community User Access (HIGH)

Guest users are unauthenticated. All classes accessible from Experience Cloud must use `with sharing`.

```apex
@AuraEnabled
public static List<Case> getMyCases() {
    String userType = UserInfo.getUserType();
    if (userType == 'Guest') {
        throw new AuraHandledException('Authentication required.');
    }
    return [SELECT Id FROM Case WHERE OwnerId = :UserInfo.getUserId() WITH SECURITY_ENFORCED];
}
```

- OWD on sensitive objects should never be Public Read/Write
- External OWD can be set separately from internal OWD

**Reference:** [Guest User Security Best Practices](https://help.salesforce.com/s/articleView?id=sf.networks_security_guest_user.htm)

---

## 13. Shield Platform Encryption (CRITICAL for Regulated Data)

Use Shield Platform Encryption for HIPAA, PCI-DSS, GDPR workloads. Classic Encryption is NOT equivalent.

| Scenario | Use |
|---|---|
| Healthcare, financial, PII at rest | Shield Platform Encryption (AES-256) |
| Simple field masking from UI users | Classic Encrypted Text field |
| Full SOC 2 / HIPAA compliance proof | Shield only |

Shield Checklist:
- [ ] Shield license confirmed before any encryption design
- [ ] Encryption policy configured in Setup before deploying encrypted fields
- [ ] SOQL reviewed -- LIKE, ORDER BY, GROUP BY don't work on encrypted fields
- [ ] Test coverage includes encrypted field handling

**Reference:** [Shield Platform Encryption Overview](https://help.salesforce.com/s/articleView?id=sf.security_pe_overview.htm)

---

## 14. Transaction Security Policies (HIGH)

Apex-based policies that monitor and block suspicious activity in real time.

```apex
global class DataExportPolicy implements TxnSecurity.EventCondition {
    public Boolean evaluate(SObject event) {
        ReportEvent re = (ReportEvent) event;
        if (re.RowsProcessed > 10000) {
            return true; // policy fires - block or notify
        }
        return false;
    }
}
```

**Reference:** [Transaction Security Policies](https://help.salesforce.com/s/articleView?id=sf.enhanced_transaction_security_policy_types.htm)

---

## 15. OWASP API Security Top 10 -- Salesforce Mapping

| OWASP API Risk | Salesforce Manifestation | Fix |
|---|---|---|
| API1: Broken Object Level Auth | Apex REST returns records without sharing check | `with sharing` + CRUD check |
| API2: Broken Authentication | Hardcoded credentials, no Named Credential | Named Credential + OAuth |
| API3: Broken Object Property Level Auth | Returning all fields including sensitive ones | `WITH SECURITY_ENFORCED` or `stripInaccessible` |
| API4: Unrestricted Resource Consumption | SOQL without LIMIT, no rate limiting | LIMIT + Queueable for bulk ops |
| API5: Broken Function Level Auth | `@AuraEnabled` method callable by guest user | UserType check + Permission checks |
| API6: Unrestricted Access to Sensitive Flows | Flow accessible to all users without bypass check | `$Permission` bypass check first element |
| API7: Server-Side Request Forgery | User-controlled endpoint in callout | Named Credential only |
| API8: Security Misconfiguration | Debug logs in prod, guest profile too open | Log levels, guest profile audit |
| API9: Improper Inventory Management | Old REST endpoints still active | Review all `@RestResource` classes |
| API10: Unsafe Consumption of APIs | No validation of external API responses | Validate all response fields before DML |

---

## Pre-Deployment Security Checklist

- [ ] No SOQL injection (bind variables or whitelisted fields only)
- [ ] All SOQL uses `WITH SECURITY_ENFORCED` or `stripInaccessible()` (except CMT/Custom Settings)
- [ ] All DML has CRUD permission checks
- [ ] All classes declare `with sharing` or `inherited sharing` (document exceptions)
- [ ] No hardcoded IDs, credentials, or API keys
- [ ] All callouts use HTTPS, Named Credentials, and have a timeout
- [ ] No `innerHTML` / `eval()` / `Function()` in LWC or Aura
- [ ] No `JSON.deserialize` with user-controlled type names
- [ ] All SOQL on non-CMT objects has a LIMIT clause
- [ ] No sensitive data (PII, credentials) in `System.debug` calls
- [ ] Guest/Community access reviewed for any `@AuraEnabled` method
- [ ] Shield Platform Encryption in use for regulated or PII fields
- [ ] Transaction Security Policies reviewed for new data export or API patterns
- [ ] OWASP API Top 10 mapped for any new REST endpoints

---

## Sources

- [Salesforce Security Guide](https://help.salesforce.com/s/articleView?id=sf.security_overview.htm)
- [Apex Secure Coding Guide](https://developer.salesforce.com/docs/atlas.en-us.secure_coding_guide.meta/secure_coding_guide/secure_coding_apex.htm)
- [SOQL Injection Prevention](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/pages_security_tips_soql_injection.htm)
- [Shield Platform Encryption](https://help.salesforce.com/s/articleView?id=sf.security_pe_overview.htm)
- [Transaction Security Policies](https://help.salesforce.com/s/articleView?id=sf.enhanced_transaction_security_policy_types.htm)
- [LWC Security](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_security_xss.htm)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [Well-Architected: Trusted](https://architect.salesforce.com/well-architected/trusted/secure)
