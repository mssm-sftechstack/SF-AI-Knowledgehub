# Architect Review: Salesforce Constraints & AI Blind Spots

This document strengthens the knowledge base with explicit constraints, execution complexity, and security risks that AI systems routinely miss.

---

## 1. Governor Limits: The Hard Ceiling

These limits are per transaction (not per request, not per hour). AI code hits these in bulk scenarios.

### Per-Transaction Limits (Most Common)

| Limit | Value | Why AI Fails | Impact |
|-------|-------|-------------|--------|
| **SOQL Queries** | 100 | AI nests queries in loops. Thinks 100 total is OK. | 101st query = Exception. No partial rollback. |
| **DML Statements** | 150 | AI generates `insert` inside loop. Common. | 151st update = Exception. Everything rolls back. |
| **CPU Time** | 10,000 ms | AI creates nested loops on large datasets. No awareness of CPU cost per operation. | Transaction aborts. User sees timeout. |
| **Heap Size** | 6 MB (sync) | AI loads entire list into memory: `List<Account> all = [SELECT ... ]`. | Out of memory exception. |
| **Callouts** | 100 | AI makes HTTP calls in a loop without batching. | Hits limit mid-transaction. |
| **Future Calls** | 50 | AI enqueues Queueable in a loop without checking depth. | Extra futures silently ignored (not an error). |

### Why AI Misses These

- **No loop awareness**: Doesn't track "how many times will this execute?"
- **No bulk mindset**: Assumes "the user only updates one record" (wrong)
- **No cost model**: Doesn't know SOQL has CPU cost per record scanned
- **No transaction boundaries**: Thinks each operation is independent

### Architect's Rule

**Before accepting AI code, ask**:
1. Does it query inside a loop? (Almost always too many queries)
2. Does it DML inside a loop? (Almost always too many DML)
3. How many records does it handle? (200 is standard bulk size)
4. What's the SOQL cost? (Count records scanned, not just returned)

---

## 2. Order of Execution: Where Hidden Complexity Lives

### The Execution Path (Synchronous, Single Transaction)

```
User saves record
    ↓
Validation Rules fire (before DML)
    ↓
Before Insert/Update Triggers
    ↓
Record committed to database (ID assigned)
    ↓
Assignment/Escalation Rules (Cases/Leads only)
    ↓
After Insert/Update/Delete Triggers
    ↓
Record-Triggered Flows fire (async, separate transaction)
    ↓
Outbound messages, emails, tasks
    ↓
User sees "save successful"
```

**Critical**: Flows run AFTER the trigger finishes, in a separate transaction. But the trigger doesn't know a Flow exists.

### Real Failure Example: Recursive Update Loop

**Scenario**: 
- **Trigger**: After update on Account, update `Stage__c` to "Active"
- **Flow**: On Account update, check if Stage = "Active", then update `LastActiveDate__c`

**What Happens**:
1. User updates Account → trigger fires
2. Trigger updates `Stage__c` = "Active"
3. Flow sees the update, runs (separate transaction)
4. Flow updates `LastActiveDate__c`
5. **Second trigger fires** (because account was updated by Flow)
6. If trigger updates again, Flow fires again...
7. **Eventually**: 10 flow instances pending, hitting async limits

**Why AI Misses It**:
- AI sees the trigger code only
- Flow is in a different system (not in Apex code)
- No visibility into "what else will update this record"

**The Fix**:
```apex
// In trigger: set a flag to prevent re-entry
public class AccountTriggerHandler {
  private static Boolean isProcessing = false;
  
  public static void handleAfterUpdate(List<Account> accounts) {
    if (isProcessing) return;  // Guard: if Flow already updated, don't run again
    
    isProcessing = true;
    try {
      // ... update logic
    } finally {
      isProcessing = false;
    }
  }
}
```

**And in Flow**:
- Add decision: "Is this a Flow-initiated update?" (check a checkbox field)
- If yes, bypass the trigger update to prevent re-entry

---

## 3. Flow Awareness: The Invisible Automaton

### What AI Doesn't Know

AI can only see code in your codebase. It cannot see:
- Flows (stored in Flow metadata, not Apex)
- Process Builder (same problem)
- Validation Rules (metadata, not code)
- Custom Actions (available only via invocation, not visible)

### Flow Behavior AI Must Account For

| Flow Type | Executes When | Visibility | Recursion Risk |
|-----------|---------------|------------|-----------------|
| Record-Triggered | After DML completes | Hidden from Apex | **HIGH**: Can re-trigger same trigger |
| Scheduled | At specified time | Hidden | Medium: Separate transaction |
| Screen | User clicks "next" | Visible | Low: User-driven |
| Autolaunched | Invoked by code | Visible only if called | High: If invoked in loop |

### The Practical Impact

When you ask AI to write a trigger, you must say:

**Bad**: "Write a trigger to update Account.TotalRevenue when opportunities change."

**Good**: "Write a trigger to update Account.TotalRevenue. There is a Flow that updates Account.Status based on TotalRevenue. Prevent re-entry."

The second version tells AI:
- Another automation exists (Flow)
- It modifies the same record (Account)
- Therefore, guard against recursive execution

---

## 4. Security & Multi-Tenant Constraints

### The Core Problem: Data Isolation

Salesforce runs 1000s of orgs on the same database and application servers. Your code must:
- Not assume you can see all records (Org-Wide Defaults apply)
- Not assume you can update all records (Sharing Rules apply)
- Not assume you can read all fields (Field-Level Security applies)

### with sharing vs without sharing

```apex
// ✅ Default: respects sharing rules
public with sharing class AccountService {
  public static List<Account> getAccounts() {
    return [SELECT Id, Name FROM Account];
    // Returns only accounts user has access to
  }
}

// ❌ Bypasses sharing (rarely justified)
public without sharing class AdminAccountService {
  public static List<Account> getAllAccounts() {
    return [SELECT Id, Name FROM Account];
    // Returns all accounts in org, ignoring sharing
  }
}
```

**Rule**: Use `with sharing` by default. Document any `without sharing` use case.

### CRUD & FLS: Not Optional

**CRUD** = Can user Create/Read/Update/Delete the object?
**FLS** = Can user Read/Edit the specific field?

Both must be enforced. AI routinely skips both.

```apex
// ❌ AI-generated: no security checks
List<Account> accounts = [SELECT Id, Name, Sensitive__c FROM Account];
update accounts;

// ✅ Correct: enforces CRUD + FLS
List<Account> accounts = [
  SELECT Id, Name, Sensitive__c 
  FROM Account 
  WITH SECURITY_ENFORCED
];

// Verify FLS on write
if (!Account.Sensitive__c.getDescribe().isUpdateable()) {
  throw new SecurityException('Field not updateable by user');
}

update accounts;
```

### Data Sensitivity Risk: A Blind Spot

**Do not paste real org data into AI tools.** This includes:
- Account names (reveals business relationships)
- Contact emails (privacy risk)
- Opportunity amounts (financial data)
- Custom fields (may contain SSN, credit card, PII)

**Why**: 
- AI tools log conversations (Anthropic, OpenAI, others)
- Training data may be used to improve models
- Regulatory risk (GDPR, CCPA, industry compliance)

**Safe Practice**:
- Sanitize: Change "Acme Corp" to "Test Company"
- Anonymize: Remove real IDs, emails, amounts
- Pseudonymize: Replace sensitive values with placeholders
- Test in sandbox only with fake data

---

## 5. Language Tightening

### Before: Generic AI Language

❌ "It's important to understand that Salesforce is a multi-tenant platform..."
❌ "This ensures that your code is robust and secure..."
❌ "One of the best practices is to always use WITH SECURITY_ENFORCED..."

### After: Architect Tone

✅ "Salesforce runs 1000s of orgs on shared servers. Your code must enforce FLS."
✅ "WITH SECURITY_ENFORCED prevents silent field-stripping when users lack access."
✅ "Use `with sharing` on every class. Document exceptions."

---

## 6. Integration with Existing Docs

### Link to ORDER_OF_EXECUTION.md
This document assumes you've read Order of Execution to understand trigger timing and Flow interaction.

### Link to SECURITY.md
For detailed FLS enforcement patterns and permission testing, see Security Guardrails.

### Link to MULTITENANT.md
For deeper understanding of pod allocation, resource contention, and why limits exist at all.

---

## Checklist: Before Accepting AI-Generated Code

- [ ] Does it query inside a loop? (Red flag)
- [ ] Does it DML inside a loop? (Red flag)
- [ ] Will it handle 200+ records without hitting SOQL/DML limits?
- [ ] Does it account for Flows that might update the same record?
- [ ] Is there a recursion guard if re-entry is possible?
- [ ] Does it use `WITH SECURITY_ENFORCED` on all SOQL?
- [ ] Does it verify FLS before writing to custom fields?
- [ ] Is the class marked `with sharing`?
- [ ] Does it handle errors without propagating exceptions in triggers?
- [ ] Is the code tested with 200+ records (bulk scenario)?
- [ ] Was the test written as non-admin user (System.runAs)?
- [ ] Have you verified no real org data was used in prompts?

---

## Key Insight

AI doesn't understand Salesforce is a multi-tenant platform with invisible automations (Flows, validation rules, processes). It optimizes for single-record, single-trigger scenarios. Your job as architect is to:

1. **Tell AI what it can't see** (Flows, other automations)
2. **Enforce the constraints** (with sharing, SECURITY_ENFORCED, FLS)
3. **Test at scale** (200+ records, bulk DML)
4. **Validate the security** (review with security team if PII involved)

This isn't a limitation of AI—it's a limitation of code context. Good prompts compensate.

