# Regulatory & Compliance Guardrails (GDPR / HIPAA)

**CRITICAL DIRECTIVE FOR AI:** You are generating code for a system that houses Personally Identifiable Information (PII) and Protected Health Information (PHI). You must assume all data is highly regulated.

## 1. The Debug Log Data Leak
Debug logs in Salesforce are stored as plain text and can be viewed by anyone with the "View All Data" permission. 
* **The Rule:** NEVER use `System.debug()` to print raw SObjects, HTTP request/response bodies, or specific field values (e.g., Email, SSN, Phone, Revenue) unless explicitly instructed for a non-production environment.
* **The Fix:** Log the *state* or the *ID*, not the payload.

```apex
// ❌ DANGEROUS: Leaks PII into plain-text logs
System.debug('Failed Contact: ' + con); 

// ✅ SECURE: Logs the error state without leaking data
System.debug(LoggingLevel.ERROR, 'Update failed for Contact ID: ' + con.Id);
```

## 2. Shield Platform Encryption Awareness
If an org uses Salesforce Shield, certain fields are encrypted at rest. AI models frequently write SOQL queries that violate Shield constraints.
* **The Rule:** You cannot use encrypted fields in a SOQL `ORDER BY` clause, `GROUP BY` clause, or aggregate functions (`MAX`, `MIN`). 
* If a user asks you to sort or group by a sensitive field (like SSN or a custom Account Number), you must warn them that this will fail if the field is encrypted.

## 3. GDPR "Right to be Forgotten" (Hard Deletions)
* When instructed to delete user data for compliance reasons, standard `delete` is insufficient as it leaves data in the Recycle Bin for 15 days.
* **The Rule:** For compliance-related deletions, explicitly use the `emptyRecycleBin` method or `Database.emptyRecycleBin()` to permanently purge the data.