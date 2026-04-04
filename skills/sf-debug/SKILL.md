---
name: sf-debug
description: >
  Salesforce debug log analysis, log levels, Developer Console, VS Code log viewer,
  System.debug best practices, common error patterns and root causes, and production
  incident troubleshooting. Based on Salesforce Developer Console and Debug Log
  documentation.
allowed-tools: Bash(sf apex run*), Bash(sf apex tail log*)
---

# Salesforce Debug Skill

**Debug log analysis, log levels, Developer Console, VS Code log viewer, common error patterns, and production troubleshooting.**

## 1. Log Levels

| Level | What It Captures |
|---|---|
| NONE | Nothing |
| ERROR | Unhandled exceptions and fatal errors only |
| WARN | Warnings + errors |
| INFO | Informational events + warnings + errors |
| DEBUG | System.debug() calls + everything above |
| FINE | More granular method entry/exit events |
| FINER | Variable assignments, callout details |
| FINEST | Maximum detail -- heap allocations, full SOQL result sets, line-by-line |

### Recommended levels by scenario

| Scenario | DB | Apex Code | Callout | SOQL | Workflow |
|---|---|---|---|---|---|
| Normal dev | INFO | DEBUG | INFO | INFO | INFO |
| Debugging logic | INFO | FINEST | INFO | FINE | INFO |
| Debugging SOQL | INFO | DEBUG | INFO | FINEST | INFO |
| Debugging callouts | INFO | DEBUG | FINEST | INFO | INFO |
| Production incident | WARN | DEBUG | WARN | WARN | WARN |

---

## 2. Setting Debug Logs via CLI

```bash
sf apex tail log --target-org <your-org-alias>
sf apex tail log --target-org <your-org-alias> --debug-level FINEST
sf apex run --file scripts/apex/myScript.apex --target-org <your-org-alias>
sf apex list log --target-org <your-org-alias>
sf apex get log --log-id <logId> --target-org <your-org-alias>
sf apex get log --number 1 --target-org <your-org-alias>
```

To set a debug log for a specific user in Setup: Setup > Debug Logs > New. Logs are retained for 24 hours.

---

## 3. Reading a Debug Log

Key event types:

| Event Type | Meaning |
|---|---|
| `SOQL_EXECUTE_BEGIN` | SOQL query started -- shows the query text |
| `SOQL_EXECUTE_END` | SOQL query finished -- shows row count returned |
| `DML_BEGIN` | DML operation started -- shows op type, object, row count |
| `DML_END` | DML operation finished |
| `USER_DEBUG` | Output from System.debug() |
| `EXCEPTION_THROWN` | Exception raised -- shows type and message |
| `CODE_UNIT_STARTED` | Entry into a class method or trigger |
| `CODE_UNIT_FINISHED` | Exit from a class method or trigger |
| `CALLOUT_REQUEST` | Outbound HTTP callout initiated |
| `CALLOUT_RESPONSE` | Outbound HTTP callout response received |
| `LIMIT_USAGE_FOR_NS` | Governor limit snapshot |

To find where an exception originated: search for `EXCEPTION_THROWN`, then look at `CODE_UNIT_STARTED` lines above it.

---

## 4. Limit Tracking Pattern

```apex
System.debug('SOQL used: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
System.debug('DML used: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());
System.debug('Heap used: ' + Limits.getHeapSize() + '/' + Limits.getLimitHeapSize() + ' bytes');
System.debug('CPU used: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime() + ' ms');
```

```apex
// dev-only -- delete before deploy
private static void logLimits(String checkpoint) {
    System.debug('LIMITS @ ' + checkpoint +
        ' | SOQL: ' + Limits.getQueries() + '/' + Limits.getLimitQueries() +
        ' | DML: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements() +
        ' | CPU: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime() + 'ms');
}
```

Never ship `Limits.*` calls.

---

## 5. Common Errors Reference

| Error Message | Root Cause | Fix |
|---|---|---|
| `System.LimitException: Too many SOQL queries: 101` | SOQL inside a for loop | Move query before the loop, use a Map keyed by Id |
| `System.NullPointerException: Attempt to de-reference a null object` | Accessing `.field` on a null SObject | Add null check |
| `FIELD_INTEGRITY_EXCEPTION` | Lookup field set to wrong-type record Id | Verify the record exists and Id type matches |
| `MIXED_DML_OPERATION` | Setup object and non-setup object in same transaction | Use `System.runAs()` in tests |
| `UNABLE_TO_LOCK_ROW` | Two transactions updating the same record | Reduce transaction scope or use Platform Events |
| `Callout from scheduled Apex not supported` | `System.schedule()` used in a test for a callout-enabled Schedulable | Test via `new MySchedulable().execute(null)` directly |
| `Entity is deleted` | Code accessed a soft-deleted record | Query `ALL ROWS` if needed |
| `List has no rows for assignment to SObject` | SOQL returned 0 rows, assigned directly to SObject | Use `List<SObject>` and check `isEmpty()` |
| `Too many email invocations: 11` | `Messaging.sendEmail()` inside a loop | Collect into list, call `sendEmail()` once |
| `Apex CPU time limit exceeded` | Heavy synchronous processing | Offload to `Queueable` or `Batch` |
| `SObject row retrieved via SOQL without querying the requested field` | Field not in SELECT | Add the field to your SOQL |
| `You cannot deploy to a required field` | A `required=true` custom field is in `fieldPermissions` in a PS | Remove that field from `fieldPermissions` |
| `Read only: field is not writeable` | Attempting to set a formula or auto-number field | Remove the field from the SObject assignment |

---

## 6. Anonymous Apex for Quick Tests

```apex
// scripts/apex/testAccountService.apex
AccountService svc = new AccountService();
List<Account> result = svc.getTopAccounts(10);
System.debug('Count returned: ' + result.size());
for (Account a : result) {
    System.debug(a.Name + ': ' + a.AnnualRevenue);
}
```

```bash
sf apex run --file scripts/apex/testAccountService.apex --target-org <your-org-alias>
```

---

## 7. Production Debug Strategy

Enable debug logging for a specific user only -- never enable org-wide FINEST in production.

1. Setup > Debug Logs > New
2. Select "User" as the traced entity type
3. Pick the specific user reporting the issue
4. Set log levels to minimum needed (DEBUG for Apex Code, WARN for everything else)
5. Reproduce as that user
6. Retrieve the log within 24 hours

```bash
sf apex tail log --target-org <your-org-alias>
sf apex get log --log-id 07Lxx000000xxxx --target-org <your-org-alias> --output-dir logs/
```

Disable the trace flag when done. Log retention: 24 hours.

---

## Sources

- [Debug Log Documentation](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_debugging_debug_log.htm)
- [Developer Console Overview](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/code_dev_console.htm)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [System.Limits Class Reference](https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_methods_system_limits.htm)
