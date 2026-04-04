---
name: sf-flow
description: >
  Salesforce Flow development skill. Covers record-triggered, screen, scheduled,
  autolaunched flows, subflows, and Flow Orchestrator. Enforces bypass logic, fault paths,
  naming conventions, security (system context awareness), performance, triggerOrder for
  deterministic execution, and XML deployment requirements.
  Based on Flow Builder Guide and Well-Architected Framework (API 62.0).
allowed-tools: Bash(sf project deploy*), Bash(sf project retrieve*)
---

# Salesforce Flow Skill

**Record-triggered, screen, scheduled, and autolaunched flows with bypass logic, fault paths, and XML deployment patterns.**

## Flow Type Selection

| Use Case | Flow Type | Performance Note |
|---|---|---|
| Update fields on triggering record | Before Save Record-Triggered | Fastest — no extra DML transaction |
| Create/update related records | After Save Record-Triggered | Runs after commit |
| Time-based automation | Scheduled Path in Record-Triggered | Part of the same flow |
| Standalone bulk scheduled job | Scheduled Flow | Separate scheduled job |
| User-facing wizard or input | Screen Flow | Runs in user context |
| Called from Apex or REST API | Autolaunched Flow | System context unless `inputVariables` override |
| Reusable logic block | Subflow | Called from parent flows |
| Multi-step, multi-user approval process | Flow Orchestrator | See section below |

**Prefer declarative (Flow) over programmatic (Apex) for automation.** Only use Apex when Flow cannot handle the complexity or volume.

**Reference:** [Flow Types Overview](https://help.salesforce.com/s/articleView?id=sf.flow_ref_types.htm)

---

## Where Flows Sit in the Order of Execution

Salesforce fires automations in a fixed sequence on every save. Flows occupy two specific positions:

| Flow type | When it fires | Step number |
|-----------|--------------|-------------|
| Before-save record-triggered | BEFORE before-triggers | Step 2 |
| After-save record-triggered | AFTER after-triggers, AFTER workflow | Step 11 |

**What this means in practice:**
- A before-save flow fires before your trigger's `before insert` / `before update` context
- An after-save flow fires after all trigger logic is complete
- If a flow updates a field AND a trigger updates the same field, the flow's change wins if it's before-save, the trigger wins if it's after-save
- Workflow field updates (step 9) restart the sequence from step 1 — flows and triggers fire again
- Platform Events published inside a flow survive a transaction rollback (they're not rolled back)

Use before-save flows for field updates (no DML cost). Use after-save flows for related record operations that need the saved record's Id.

See [sf-apex](../sf-apex/) for the full 17-step order of execution sequence.

---

## Critical Rule 1 — Flow System Context and Security Implications

Record-triggered flows and autolaunched flows run in **system context by default** — they bypass FLS, CRUD, and sharing rules. This is a major security implication.

```
Before Save flow    → system context (no sharing, no FLS)
After Save flow     → system context
Scheduled flow      → system context
Screen flow         → user context (respects sharing and FLS)
Autolaunched flow   → system context (unless called from user-context Apex with sharing)
```

Design implications:
- Never expose system-context flow data back to the user without filtering
- Be explicit in design docs that a flow runs as system
- Screen Flows use the running user's context — validate input carefully
- Use `$Permission` checks in flows to restrict which users can trigger sensitive logic

**Reference:** [Flow Security](https://architect.salesforce.com/well-architected/trusted/secure)

---

## Critical Rule 2 — Bypass Logic (FIRST element always)

Every record-triggered flow MUST start with a bypass check:
```
Decision: "Check Bypass"
  Condition: {!$Permission.Bypass_Flows} = TRUE → [END]
  Default: "Continue" → [rest of flow]
```
Without this, data migrations and integrations trigger thousands of unwanted side effects.
Create a single `Bypass_Flows` Custom Permission — assign it to the migration/integration user's Permission Set.

**Reference:** [Custom Permissions in Flow](https://help.salesforce.com/s/articleView?id=sf.flow_ref_operators_global.htm)

---

## Critical Rule 3 — Fault Paths on Every Data Element

Every Get Records, Create Records, Update Records, Delete Records element MUST have a Fault connector:
```
Get Records → Success path → [next element]
           → Fault path   → Assignment: store {!$Flow.FaultMessage} → Screen/Notification/END
```
Flows without fault paths silently fail and give users no feedback.

**Reference:** [Flow Fault Paths](https://help.salesforce.com/s/articleView?id=sf.flow_ref_elements_fault.htm)

---

## Critical Rule 4 — XML Deployment Requirements

```xml
<status>Active</status>        <!-- REQUIRED: flows deploy as Inactive without this -->
<apiVersion>62.0</apiVersion>  <!-- REQUIRED: always match project API version -->
```

**Reference:** [Flow Metadata Reference](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_visual_workflow.htm)

---

## Critical Rule 5 — triggerOrder for Deterministic Execution

When multiple record-triggered flows fire on the same object and event, their execution order is undefined unless `triggerOrder` is set. Always set it explicitly:

```xml
<triggerOrder>1</triggerOrder>   <!-- lower number runs first -->
```

- Use values like 1, 10, 20, 100 — leave gaps for future flows to be inserted between
- Document the trigger order in the flow description
- Audit all flows on an object and assign explicit trigger orders when deploying to orgs with existing flows

**Reference:** [Flow Trigger Ordering](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_trigger_ordering.htm)

---

## Order of Execution — Where Flows Fit

Flows run AFTER Apex triggers in the Salesforce Order of Execution. Key implications:

```
1. System Validation
2. Before-Save Flow (Before Save Record-Triggered)   ← runs here
3. Before Triggers (Apex)
4. System Validation Rules
5. Custom Validation Rules
6. Duplicate Rules
7. Record saved to database (not committed)
8. After Triggers (Apex)
9. Assignment Rules
10. Auto-Response Rules
11. Workflow Rules
12. Escalation Rules
13. After-Save Flow (After Save Record-Triggered)    ← runs here
14. Entitlement Rules
15. Roll-Up Summary recalculation
16. Criteria-based Sharing
17. Commit to database
```

- Before Save Flows run before Apex triggers — use them to set fields without triggering a re-evaluation
- After Save Flows run after all Apex triggers — they see the committed record state
- Flows and Apex triggers together can create recursive loops — design carefully

**Reference:** [Order of Execution](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)

---

## Transaction Limits in Flows

Flows are subject to Apex governor limits per transaction:
- Max 150 DML statements per transaction
- Max 100 SOQL queries per transaction
- Before Save flows: limited to 1 DML operation on the triggering record only (no related records)
- After Save flows: can create/update related records but each path counts toward DML limits
- Scheduled flows: run in their own transaction per batch of 200 records

```
Before Save: update trigger record fields → FREE (no DML counted)
After Save: create child records → counts toward DML limit
```

Too many Get Records in a loop will hit SOQL limits — use collections instead.

**Reference:** [Flow Governor Limits](https://help.salesforce.com/s/articleView?id=sf.flow_considerations.htm)

---

## Flow Orchestrator — Multi-User, Multi-Step Workflows

Use Flow Orchestrator when automation requires multiple users to complete steps in sequence or parallel — e.g. multi-stage approvals, onboarding checklists, multi-team handoffs.

### When to Use Flow Orchestrator vs Approval Process
| Scenario | Use |
|---|---|
| Simple one-level approval with approve/reject | Standard Approval Process |
| Multi-step, multi-user workflow with conditionals | Flow Orchestrator |
| Different users at each stage | Flow Orchestrator |
| Steps that can run in parallel | Flow Orchestrator |
| Custom screen UI between steps | Flow Orchestrator (with Screen Flows as steps) |

### Orchestrator Anatomy
```
Orchestration (the parent, auto-launched)
  ├── Stage 1: Review          (Interactive — assigned to Manager role)
  │     └── Step: Approve Budget (Screen Flow — user completes)
  ├── Stage 2: Legal Sign-off  (runs after Stage 1 completes)
  │     └── Step: Legal Review  (Screen Flow — assigned to Legal queue)
  └── Stage 3: Finance Approval (Background — auto-processes, no user input)
        └── Step: Update Records (Autolaunched Flow)
```

### Setup Notes
- Orchestrations require "Orchestration" Flow type in metadata
- Each stage can be sequential or parallel
- Background steps run immediately; Interactive steps wait for user action
- Use `$Orchestration` global variable to access orchestration context
- Assign steps to Users, Queues, or Roles — not to groups

**Reference:** [Flow Orchestrator Overview](https://help.salesforce.com/s/articleView?id=sf.flow_orchestration_overview.htm)

---

## Naming Conventions

**Flow Label:** `[Action] [Object]: [Type]`
- `Send Closed Won Alert: Opportunity – After Save`
- `Assign Territory: Account – Before Save`
- `Onboarding Wizard: Contact – Screen`

**API Name:** same in underscored format
- `Send_Closed_Won_Alert_Opportunity_After_Save`

**Variable naming:**

| Type | Pattern | Example |
|---|---|---|
| Record | `var{Object}` | `varOpportunity` |
| Collection | `col{Object}` | `colContacts` |
| Text | `txt{Purpose}` | `txtEmailBody` |
| Boolean | `bool{Condition}` | `boolIsEligible` |
| Number | `num{Purpose}` | `numRetryCount` |
| ID | `id{Purpose}` | `idAccountOwner` |

**Element descriptions:** MANDATORY on every Decision, Assignment, Get/Update/Create/Delete Records, and variable declaration.

**Reference:** [Flow Builder Best Practices](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_best_practices.htm)

---

## Performance Rules

- Never Get Records inside a loop — query once outside, filter with a Decision
- Prefer Before Save for field updates on the triggering record (no DML consumed)
- Use entry criteria to prevent firing on irrelevant updates
- Max recommended elements per flow: 2,000 — split into subflows beyond this
- For bulk scheduled flows, Salesforce processes up to 200 records per batch — design idempotently
- Set `triggerOrder` explicitly when multiple flows exist on the same object+event

**Reference:** [Flow Performance Optimization](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_bulkification.htm)

---

## Screen Flow Security

- Always validate user input in Screen Flows — malicious data can reach your Apex actions
- Don't expose internal system fields in screen flow output — filter before displaying
- Use `$User` and `$Permission` variables to conditionally show sensitive screens
- Test screen flows with minimum-permission profiles, not just admin

**Reference:** [Flow Security Considerations](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_security.htm)

---

## Flow XML Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Flow xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <description>DESCRIBE BUSINESS PURPOSE AND TRIGGER CONDITION HERE</description>
    <label>Send Closed Won Alert: Opportunity – After Save</label>
    <status>Active</status>
    <processType>AutoLaunchedFlow</processType>
    <triggerType>RecordAfterSave</triggerType>
    <objectType>Opportunity</objectType>
    <triggerOrder>10</triggerOrder>

    <start>
        <locationX>50</locationX>
        <locationY>0</locationY>
        <connector>
            <targetReference>Check_Bypass</targetReference>
        </connector>
        <filterLogic>and</filterLogic>
        <!-- entry criteria: only fire when stage changes to Closed Won -->
        <filters>
            <field>StageName</field>
            <operator>EqualTo</operator>
            <value><stringValue>Closed Won</stringValue></value>
        </filters>
        <recordTriggerType>Update</recordTriggerType>
        <scheduledPaths>
            <maxBatchSize>200</maxBatchSize>
        </scheduledPaths>
    </start>

    <!-- Bypass Decision — always first element after start -->
    <decisions>
        <name>Check_Bypass</name>
        <label>Check Bypass</label>
        <description>Exits if the Bypass_Flows Custom Permission is assigned.</description>
        <defaultConnectorLabel>Continue</defaultConnectorLabel>
        <rules>
            <name>Bypass_Active</name>
            <conditionLogic>and</conditionLogic>
            <conditions>
                <leftValueReference>$Permission.Bypass_Flows</leftValueReference>
                <operator>EqualTo</operator>
                <rightValue><booleanValue>true</booleanValue></rightValue>
            </conditions>
            <label>Bypass Active</label>
            <!-- no connector = END -->
        </rules>
    </decisions>

    <!-- add flow logic elements below the bypass decision -->

</Flow>
```

---

## Approval Processes

Approval Processes are a native Salesforce feature for structured, auditable approval workflows. They are separate from Flow and have distinct capabilities.

### Approval Process vs Flow Orchestrator vs CPQ Approvals

| Scenario | Use |
|---|---|
| Simple one-level approve/reject (e.g. Discount > 10% needs manager sign-off) | **Approval Process** |
| Approve/reject with email notification and Chatter post | **Approval Process** |
| Multi-step sequential or parallel user tasks with Screen Flow UI | **Flow Orchestrator** |
| Conditional branching between approval steps based on field values | **Flow Orchestrator** |
| Quote-level approvals with line item conditions | **CPQ Approvals** (if CPQ installed) |
| Steps assigned to queues that can be re-assigned by members | **Approval Process** (queues supported) |

Use the simplest tool that meets the requirement. Approval Processes handle the majority of enterprise approval needs without code.

**Reference:** [Approval Processes](https://help.salesforce.com/s/articleView?id=sf.approvals_landing.htm)

---

### Approval Process Anatomy

```
Approval Process
  ├── Entry Criteria         — which records enter (formula or filter)
  ├── Approver Field         — which field/user/queue receives the request
  │
  ├── Initial Submission Actions
  │     ├── Field Update     — e.g. set Status = "Pending Approval"
  │     ├── Email Alert      — notify approver
  │     └── Outbound Message — trigger external system
  │
  ├── Approval Steps (1 or more, sequential or parallel)
  │     ├── Step 1: Line Manager   → Approve/Reject
  │     ├── Step 2: Finance        → Approve/Reject (only if Step 1 approved)
  │     └── Final Approval Actions (all steps approved)
  │           ├── Field Update     — e.g. set Status = "Approved"
  │           └── Email Alert      — notify submitter
  │
  └── Rejection Actions (any step rejected)
        ├── Field Update           — e.g. set Status = "Rejected"
        └── Email Alert            — notify submitter with reason
```

---

### Record Locking During Approval

When a record is submitted for approval, **Salesforce locks it by default**:
- Standard users cannot edit the record while it is pending approval
- Admins and users with the "Modify All" permission can still edit locked records
- Unlocking is automatic when the record is approved, rejected, or recalled

Design implication: if users need to add information after submission, either:
1. Unlock the record via the Approval Process settings (allow submitter to edit), or
2. Use a related object (Note, Task) instead of the main record for supplementary info

**Reference:** [Record Locking in Approval Processes](https://help.salesforce.com/s/articleView?id=sf.approvals_locked.htm)

---

### Delegated Approvers

Approvers can designate a delegate who can approve on their behalf:
- Users set their delegate in Personal Settings → Approver Settings
- Delegates have the same approval authority as the original approver
- Delegation is per-user, not configurable in the Approval Process itself
- Email notifications go to both the approver and the delegate

---

### Querying Approval State in Apex

Use `ProcessInstance` and `ProcessInstanceWorkitem` to query approval history and pending approvals:

```apex
// find all pending approvals for a specific record
List<ProcessInstanceWorkitem> pending = [
    SELECT Id, ActorId, ProcessInstance.Status, ProcessInstance.TargetObjectId
    FROM   ProcessInstanceWorkitem
    WHERE  ProcessInstance.TargetObjectId = :opportunityId
    AND    ProcessInstance.Status = 'Pending'
    WITH   SECURITY_ENFORCED
];

// check if a record is currently pending approval
Boolean isPending = !pending.isEmpty();
```

```apex
// programmatic approval submission from Apex
Approval.ProcessSubmitRequest req = new Approval.ProcessSubmitRequest();
req.setComments('Submitting for approval via automation.');
req.setObjectId(opportunityId);
req.setNextApproverIds(new List<Id>{ managerId }); // override approver if needed

Approval.ProcessResult result = Approval.process(req);
if (result.isSuccess()) {
    // record is now pending approval
}
```

```apex
// programmatic approval or rejection from Apex (e.g. from an integration)
Approval.ProcessWorkitemRequest approve = new Approval.ProcessWorkitemRequest();
approve.setWorkitemId(workitemId);
approve.setAction('Approve'); // or 'Reject', 'Reassign'
approve.setComments('Auto-approved by integration — all conditions met.');

Approval.ProcessResult result = Approval.process(approve);
```

**Reference:** [Approval Process Apex Methods](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_process_approval.htm)

---

### Recall Actions

Submitters can recall a pending approval (if allowed in the process settings):
- Recall removes the record from the approval queue
- Recall Actions fire when a record is recalled — use them to reset Status field
- Always include a Recall Action that resets `Status__c` to its pre-submission value

---

### Approval Process Security Considerations

- Approval step actions (field updates, email alerts) run in **system context** — they bypass FLS
- Field updates in approval processes override sharing rules — test with minimum-permission users
- The `ProcessInstanceWorkitem.ActorId` can be a User or a Queue — handle both in Apex
- Chatter approval requires Chatter to be enabled — don't assume it is in all orgs

---

### Approval Process Checklist

- [ ] Entry criteria defined — only intended records enter the process
- [ ] Initial Submission Actions set Status to a "Pending" value (users know it's in review)
- [ ] Recall Actions reset Status to pre-submission value
- [ ] Final Approval and Rejection Actions both send email notifications to submitter
- [ ] Record locking behaviour reviewed — decide if submitter can edit while pending
- [ ] Delegated approver behaviour tested
- [ ] Approval process tested with minimum-permission profile (not admin)
- [ ] Programmatic submission uses `Approval.ProcessSubmitRequest` (not DML)
- [ ] If querying approval state: use `ProcessInstanceWorkitem` with `SECURITY_ENFORCED`

---

## Deployment (Confirm with User First)
```bash
sf project deploy start --metadata Flow:Send_Closed_Won_Alert_Opportunity_After_Save --target-org <your-org-alias>
sf project retrieve start --metadata Flow:Send_Closed_Won_Alert_Opportunity_After_Save --target-org <your-org-alias>
```

**Before deploying a new Flow version:** retrieve the current active version, deactivate it in the org, then deploy the new version to avoid duplicate active flow versions.

---

## Flow Quality Scoring (Ship only at 88+/110)

Score every Flow before deploying. Minimum threshold: 88/110 (80%).

| Dimension | Points | Failing Condition |
|---|---|---|
| Bulkification | 25 | Get Records inside a Loop; Create/Update inside a Loop without collection variable accumulation |
| Entry Criteria | 20 | Record-triggered flow fires on every save with no entry condition; no bypass Custom Permission check |
| Naming | 20 | Flow API name doesn't follow [Object]_[Trigger]_[Purpose]; elements have auto-generated names (Screen_1, Decision_2) |
| Fault Handling | 20 | Any Get/Create/Update/Delete element has no fault path; fault path goes nowhere (no log, no event) |
| Performance | 15 | Loop used where Transform element handles it (use Transform for data mapping — 30-50% faster); Scheduled flow volume × frequency not validated against limits |
| Documentation | 10 | No flow description set; no element description on Decision/Loop elements; no version notes |
| **TOTAL** | **110** | |

**Score 88-110** → ready to deploy
**Score 70-87** → fix failing dimensions first
**Score below 70** → reject, redesign

### Transform Over Loop — Key Rule
Use the **Transform** element instead of a Loop + Assignment for data mapping. Transform is 30-50% more performant because it avoids governor limit consumption on each iteration.

```
// When to use Transform (not Loop):
// - Map one collection to another (e.g. Accounts → custom sObjects)
// - Extract a field from a collection into a new collection
// - Combine/reshape record collections

// When Loop is still needed:
// - Conditional logic per record (if/else inside the loop)
// - Chained decisions within the iteration
// - Accumulating a count or sum
```

---

## Official Sources
- [Flow Builder Guide](https://help.salesforce.com/s/articleView?id=sf.flow.htm)
- [Flow Types](https://help.salesforce.com/s/articleView?id=sf.flow_ref_types.htm)
- [Flow Governor Limits](https://help.salesforce.com/s/articleView?id=sf.flow_considerations.htm)
- [Flow Trigger Ordering](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_trigger_ordering.htm)
- [Flow Orchestrator](https://help.salesforce.com/s/articleView?id=sf.flow_orchestration_overview.htm)
- [Fault Paths](https://help.salesforce.com/s/articleView?id=sf.flow_ref_elements_fault.htm)
- [Flow Metadata Reference](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_visual_workflow.htm)
- [Order of Execution](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)
- [Well-Architected: Intentional (Declarative First)](https://architect.salesforce.com/well-architected/easy/intentional)
- [Well-Architected: Composable](https://architect.salesforce.com/well-architected/adaptable/composable)
- [Well-Architected: Secure (Flow Context)](https://architect.salesforce.com/well-architected/trusted/secure)
