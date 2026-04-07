# Order of Execution in Salesforce

This document explains when triggers, flows, validation rules, and async processes execute. Understanding execution order is critical to avoid conflicts and race conditions.

---

## Synchronous Execution Order (Same Transaction)

### Before DML (Save)

1. **Validation Rules** — Run against all records
   - If any validation rule fails, the transaction stops
   - DML never happens

2. **Before-Insert/Update Triggers** — Custom Apex logic
   - `trigger MyTrigger on Account (before insert) { }`
   - Can modify field values before save
   - Cannot update related objects (they haven't been saved yet)

### During DML

3. **DML Commits** — Records insert/update/delete
   - If successful, records now have IDs
   - Triggers move to after-insert/update/delete phase

### After DML

4. **Assignment Rules** — Auto-assign cases or leads
5. **Escalation Rules** — Escalate cases based on conditions
6. **After-Insert/Update/Delete Triggers** — Custom Apex logic after records exist
   - Can see new field values and IDs
   - Can perform DML on related objects
   - Cannot modify the primary record being inserted/updated (not in after-trigger context)

7. **Record-Triggered Flows** — Execute after DML
   - Can read and update related records
   - Run in parallel (not guaranteed order if multiple flows on same object)

### Workflow Notifications & Actions

8. **Outbound Messages** — Sent to external systems
9. **Email Alerts** — Sent to specified recipients
10. **Task Creation** — Assigned to specified users

---

## Asynchronous Execution (Different Transaction)

After the primary transaction completes, async jobs start in their own transactions:

### Platform Events

```apex
// Published in main transaction
EventBus.publish(new Custom_Event__e(field = value));

// Subscribed flow runs AFTER main transaction commits
flow: Event-triggered flow → updates records in separate transaction
```

### Scheduled Apex

```apex
System.schedule('Job Name', '0 0 0 * * ?', new MySchedulableClass());
```

- Runs at specified time (not tied to any DML)
- Separate transaction with own governor limits
- No callout from synchronous context, but Queueable called from Scheduled Apex can callout

### Batch Apex

```apex
Database.executeBatch(new MyBatchClass(), 200);
```

- Executes in chunks (batch size)
- Each batch is a separate transaction with separate governor limits
- Can run for hours or days

### Queueable

```apex
System.enqueueJob(new MyQueueableClass());
```

- Executes soon after enqueue (within seconds, not guaranteed immediate)
- Can chain into itself for sequential processing
- Can callout with `Database.AllowsCallouts` interface
- Each enqueue is a separate transaction

---

## Gotchas & Race Conditions

### Gotcha 1: Trigger + Flow Conflict

**Scenario**: Account trigger updates `Revenue__c`, Flow also updates `Revenue__c`

```apex
// Trigger code
trigger AccountTrigger on Account (after update) {
  for (Account acc : Trigger.new) {
    acc.Revenue__c = 50000;  // Tries to update
  }
  update Trigger.new;
}

// Flow: Account → Update Record → Revenue__c = 50000
```

**Execution Order**:
1. Before trigger fires
2. DML commits
3. After trigger fires (updates to 50000)
4. Flow runs (updates to 50000 again)
5. **Problem**: Order not guaranteed, potential race condition. Last write wins.

**Fix**: Remove from one location. Use trigger XOR flow, not both.

### Gotcha 2: Before-Insert Dependencies

**Cannot query records you're inserting**:

```apex
// ❌ Wrong
trigger OpportunityTrigger on Opportunity (before insert) {
  List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE Id IN :Trigger.new];  // Empty! Opps don't exist yet
}

// ✅ Right
trigger OpportunityTrigger on Opportunity (before insert) {
  for (Opportunity opp : Trigger.new) {
    opp.IsClosed = false;  // Modify field before insert
  }
}
```

### Gotcha 3: Subflow Recursion

**Flows can call flows recursively, leading to infinite loops**:

```
Flow A → Calls Subflow B → Subflow B updates record → Record-triggered Flow A fires again
```

**Fix**: 
- Add a checkbox field `Block_Flow_Recursion__c`
- In Flow A, check if already processed: Decision "Is_Block_Flow_Recursion__c true?" → If yes, exit
- Set `Block_Flow_Recursion__c = true` at start of Flow A

### Gotcha 4: Rollup Summary Infinite Loop

**Rollup Summary fields can trigger after-insert/update flows, which update child records, which trigger rollup again**:

```
Parent record updates → Rollup Summary recalculates → Updates parent field → After-update trigger fires → Updates child
→ Rollup Summary recalculates again
```

**Fix**: 
- Never update a child in an after-update trigger if a Rollup Summary points to that field
- If needed, use a one-time flag to prevent recursion

### Gotcha 5: Validation Rules After After-Trigger DML

**Validation rules run on the initial record, not on DML from after-trigger**:

```apex
// Account record inserted with Name = "Test"
// After-trigger fires and updates Name = "" (invalid per validation rule)
update accounts;  // ⚠️ Validation rule does NOT block this because it already ran before trigger
```

**Fix**: Add manual validation in after-trigger or use a custom error throw.

---

## Decision Tree: When to Use What

### Should I use a Trigger or Flow?

| Scenario | Use Trigger | Use Flow | Reason |
|----------|-------------|----------|--------|
| Simple field update on save | ❌ | ✅ | Flow is visual, easier to maintain |
| Complex business logic (>100 lines) | ✅ | ❌ | Apex scales better, easier to test |
| Update related object | ✅ | ✅ | Both work; flow is simpler if available |
| Callout needed | ✅ (Queueable) | ❌ | Flows can't callout; use Queueable from trigger |
| Need subflow | ❌ | ✅ | Flows have subflow support |
| Need loops | ✅ | ✅ | Both support loops; Apex is more efficient |
| Formula-based calculation | ❌ | ✅ | Flow decisions are formula-based, clearer |
| 200+ records bulk-processed | ✅ | ⚠️ | Triggers scale, flows may hit limits |

---

## Best Practices

✅ **Do**:
- Keep triggers lean. Delegate to service classes.
- Use flows for simple, frequently-changing logic.
- Use one trigger per object (extend the handler, don't create multiple triggers).
- Always include after-trigger logic to verify DML succeeded.
- Test both trigger and flow order if both are present.

❌ **Don't**:
- Put SOQL/DML in loops in triggers.
- Create multiple triggers on the same object.
- Assume flow and trigger order is guaranteed.
- Update the same field in both trigger and flow.
- Recursively call flows without a guard flag.

---

## Testing Order of Execution

```apex
@IsTest
private class OrderOfExecutionTest {
  @IsTest
  static void testTriggerBeforeFlow() {
    // Setup
    Account acc = new Account(Name = 'Test', Revenue__c = 0);
    
    // DML in test
    insert acc;
    
    // Query to see final state (after trigger and flow)
    Account updatedAcc = [SELECT Revenue__c FROM Account WHERE Id = :acc.Id];
    
    // Assert final state (reflects trigger + flow execution)
    System.assertEquals(50000, updatedAcc.Revenue__c, 'Both trigger and flow updated');
  }
}
```

---

## Quick Reference Table

| Component | Runs | Timing | Can Update Record | Can Callout |
|-----------|------|--------|-------------------|------------|
| Validation Rule | Before trigger | Immediate | ❌ No (blocks) | ❌ No |
| Before Trigger | Before DML | Immediate | ✅ Yes | ❌ No |
| After Trigger | After DML | Immediate | ❌ No (but can update related) | ❌ No (use Queueable) |
| Flow (record-triggered) | After DML | Immediate | ✅ Yes | ❌ No |
| Queueable | Later | Async | ✅ Yes (separate transaction) | ✅ Yes |
| Scheduled Apex | Later | Scheduled time | ✅ Yes (separate transaction) | ❌ No (but can enqueue Queueable) |
| Batch Apex | Later | In chunks | ✅ Yes (separate transaction) | ❌ No (but can enqueue Queueable) |
| Platform Events | Later | Event-triggered flow | ✅ Yes (separate transaction) | ❌ No |

