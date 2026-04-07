# Flow Best Practices

Flows are powerful declarative automation tools. This guide covers best practices, common mistakes, and when to use Flow vs. Apex vs. Process Builder.

---

## When Flow Is the Right Tool

### Use Flow When

- **Logic is simple and visual** — Business logic that fits in 20 decisions or fewer
- **Logic changes frequently** — Admins need to update without code deploy
- **Subflows are needed** — Orchestrating multiple flows, breaking logic into reusable pieces
- **Error recovery is needed** — Fault paths make error handling explicit
- **Formula conditions are sufficient** — Decision logic can be expressed in formulas
- **No callouts needed** — Pure data manipulation and logic
- **Record-triggered on insert/update** — Declarative automation on save
- **Screen flows for user input** — Multi-step screens, dynamic form building

### Use Apex (Trigger) When

- **Logic is complex** — More than 50 lines of business logic
- **Callouts needed** — API calls to external systems (use Queueable from trigger)
- **Performance is critical** — Bulk operations on 200+ records (Apex scales better)
- **Reusable service logic needed** — Code that multiple triggers/classes share
- **Detailed logging needed** — Audit trail of why decisions were made
- **99%+ test coverage required** — Harder to test flows than Apex

### Don't Use Process Builder

Process Builder is deprecated. Use Flows instead.

---

## Flow Types & Their Constraints

### 1. Record-Triggered Flow

**When**: Fires when a record is inserted/updated/deleted

```
Trigger: Account insert → Record-Triggered Flow fires
Flow logic: Update related records, send emails, create tasks
```

**Constraints**:
- Cannot start from a screen
- Runs in parallel (if 10 accounts inserted, 10 flow instances run simultaneously)
- Cannot call other record-triggered flows (risk of infinite recursion)
- Cannot query/filter starting records — receives all

**Gotcha**: If 1,000 accounts insert, 1,000 flow instances spin up. If each loops through 10 opportunities, you hit limits fast.

**Fix**: Use bulk testing (see below).

### 2. Screen Flow

**When**: User-driven multi-step form

```
1. Name and Email screen
2. Decision: Is email valid?
3. Confirmation screen
4. Submit → Create Contact
```

**Constraints**:
- Cannot be invoked from flows automatically (only from Lightning UI)
- Cannot start from scheduled apex
- Timeout: 30-minute inactivity closes the flow

### 3. Scheduled Flow

**When**: Runs at a specific time

```
Scheduled: Every night at 2 AM
Action: Check all Opportunities due in 7 days, send reminders
```

**Constraints**:
- Must be activated to run
- No dynamic scheduling (time is fixed)
- Runs in its own transaction

### 4. Autolaunched Flow (Invocable)

**When**: Called from another flow, Apex, or API

```apex
// Invoked from Apex
Flow.Interview.CalculateAnnualSpendFlow interview = new Flow.Interview.CalculateAnnualSpendFlow(inputs);
interview.start();
```

**Constraints**:
- Must be explicitly started
- Can be passed input variables
- No visual/screen elements

---

## Subflows: Variable Passing & Scope

### Basic Subflow Call

```xml
<!-- Main Flow: AccountUpdate -->
<flow:definition>
  <!-- Call subflow -->
  <actionCall type="subflow">
    <label>Calculate Annual Spend</label>
    <name>CalculateSpendSubflow</name>
    <inputParameters>
      <paramName>accountId</paramName>
      <paramValue>{!accountRecord.Id}</paramValue>
    </inputParameters>
    <outputParameters>
      <paramName>totalSpend</paramName>
      <paramValue>{!totalSpend}</paramValue>
    </outputParameters>
    <faultPath>
      <connector>HandleError</connector>
    </faultPath>
  </actionCall>
</flow:definition>

<!-- Subflow: CalculateSpendSubflow -->
<flow:definition>
  <inputVariable name="accountId" type="id"/>
  <outputVariable name="totalSpend" type="number"/>
  
  <!-- Get Opportunities and sum amounts -->
  <recordLookup>
    <inputParameters>
      <filterCondition>
        <field>AccountId</field>
        <value>{!accountId}</value>
        <operator>equals</operator>
      </filterCondition>
    </inputParameters>
    <outputVariable>opportunities</outputVariable>
  </recordLookup>
</flow:definition>
```

### Best Practices for Subflows

✅ **Do**:
- Pass only the data needed (e.g., `accountId`, not entire record)
- Always include fault paths in subflow calls
- Name output variables clearly (e.g., `totalSpend`, not `result`)
- Log subflow errors to admin
- Test subflows independently

❌ **Don't**:
- Pass entire record objects (wastes memory)
- Assume subflows will succeed (always add fault paths)
- Create deeply nested subflows (hard to debug)
- Pass sensitive data in variable names (shows in logs)
- Call the same subflow recursively without guard

### Recursion Prevention

```
1. In main flow, set a flag: Set Recursion_Guard = true
2. In record-triggered flow, decision: Is Recursion_Guard = true?
3. If yes, exit. If no, proceed.
```

This prevents infinite loops when subflows update records that trigger the parent flow again.

---

## Fault Paths & Error Recovery

Every subflow call should have a fault path:

```xml
<actionCall type="subflow">
  <label>Calculate Spend</label>
  <name>CalculateSpendSubflow</name>
  <faultPath>
    <connector>HandleError</connector>  <!-- Runs if subflow fails -->
  </faultPath>
</actionCall>

<!-- Error handler -->
<actionCall type="sendEmail">
  <label>HandleError</label>
  <inputParameters>
    <subject>Flow Error: {!$Flow.FaultMessage}</subject>
    <body>Error details: {!$Flow.FaultMessage}</body>
    <recipientList>admin@company.com</recipientList>
  </inputParameters>
</actionCall>
```

**Fault Variables** (always available):
- `{!$Flow.FaultMessage}` — Error message
- `{!$Flow.FaultStack}` — Stack trace (if available)

### Retry Pattern with Fault Paths

```
Try subflow:
- If succeeds → Continue
- If fails → Save error log + Send email to admin
- Admin manually retries or investigates
```

Don't auto-retry in flows. Let admins decide. Use Queueable for auto-retry.

---

## Platform Events & Flows

Flows can subscribe to Platform Events:

```xml
<flow:definition>
  <trigger type="event">
    <eventName>Custom_Event__e</eventName>
  </trigger>
  
  <!-- Event-triggered flow runs here -->
  <actionCall type="updateRecord">
    <label>Update Account</label>
  </actionCall>
</flow:definition>
```

**Constraint**: Runs asynchronously. Not tied to original transaction.

**Use for**: Decoupled event-driven updates (account updated → event published → separate flow updates related records).

---

## Flow Orchestrator Patterns

Flow Orchestrator orchestrates multiple flows:

```
Flow Orchestrator:
├── Start
├── Stage 1: Approval Flow
├── Stage 2: Record Creation Flow
├── Stage 3: Notification Flow
└── End
```

**Key Points**:
- Stages run sequentially (or in parallel if configured)
- State persists between stages
- Orchestrator tracks which stage completed
- Good for multi-phase workflows (approval → creation → notification)

---

## Performance Limits & Gotchas

### Governor Limits in Flows

| Limit | Value |
|-------|-------|
| SOQL queries per flow | 100 |
| Records returned per query | 50,000 |
| Loop iterations | 10,000 |
| Time limit | 10 minutes |
| Decision depth | No hard limit, but performance degrades |

### Gotcha 1: Loop Hitting Limit

```
❌ Wrong:
Loop through Accounts (1,000 items)
  → Inside loop, update Opportunities (1,000 updates in the loop)
  → Hit DML limit (10,000 records per transaction)

✅ Right:
Loop once, collect IDs
Query Opportunities in bulk
Update all in one step (outside loop)
```

### Gotcha 2: SOQL in Loops

```
❌ Wrong:
Loop through 100 Accounts:
  Query Opportunities WHERE AccountId = this account
  (100 queries hit limit)

✅ Right:
Query Opportunities WHERE AccountId IN all account IDs (1 query)
Loop and process in memory
```

### Gotcha 3: Missing Variable Initialization

```
❌ Wrong:
Decision: Is totalAmount > 100?
(totalAmount is null, decision fails)

✅ Right:
Initialize totalAmount = 0
Loop through records
Update totalAmount
Then decision
```

---

## Common Mistakes & Fixes

### Mistake 1: Infinite Loop with Recursion

**Problem**: Flow updates record → Record-triggered flow runs → Updates record again → Loop

**Fix**:
1. Add `Processed__c` checkbox field
2. In flow: Decision "Is Processed__c true?" → If yes, exit
3. Set `Processed__c = true` at start
4. Set `Processed__c = false` when done (if not marking as processed permanently)

### Mistake 2: Missing Fault Paths

**Problem**: Subflow fails silently. No error logged.

**Fix**: Always add fault paths. Log errors.

```xml
<actionCall type="subflow">
  <faultPath>
    <connector>LogError</connector>
  </faultPath>
</actionCall>
```

### Mistake 3: Assuming Record Exists

**Problem**: Record-triggered flow expects record exists, but deletes or filters might run first.

**Fix**: Always check if record exists before updating.

### Mistake 4: Variable Name Typos

**Problem**: Decision references `{!totalSpend}` but variable is `{!total_Spend}` (underscore).

**Fix**: Use copy-paste for variable names. Double-check declarations.

### Mistake 5: Time-Based Delays

**Problem**: Using a scheduled flow for near-real-time updates (not reliable).

**Fix**: Use record-triggered flow or Queueable for immediate updates.

---

## Testing Flows

### Manual Testing

1. **Test happy path**: Insert/update record with valid data
2. **Test edge cases**: Null values, empty lists, division by zero
3. **Test fault paths**: Deliberately fail subflows to verify error handling
4. **Test subflow isolation**: Test subflow separately before testing parent flow

### Assertion Patterns

- Query records after flow runs
- Assert field values match expected outcome
- Assert related records created/updated

### Bulk Testing

```
1. Bulk insert 200 accounts
2. Record-triggered flow fires 200 times (parallel)
3. Assert all related opportunities updated
4. Assert no SOQL/DML limit errors
```

---

## Checklist: Flow Ready for Production

- ✅ All subflows have fault paths
- ✅ Fault paths log errors
- ✅ No infinite recursion (guard flag in place)
- ✅ Bulk tested (200+ records)
- ✅ SOQL queries counted (< 100 per flow)
- ✅ Variable names checked for typos
- ✅ Decision logic handles null values
- ✅ Output variables named clearly
- ✅ No time-based delays (rely on record-triggered)
- ✅ Error handling in place (emails to admin on failure)

