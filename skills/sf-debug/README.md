# Salesforce Debug Skill

**Debug log analysis, log levels, Developer Console, VS Code log viewer, common error patterns, and production troubleshooting.**

## 1. Log Levels

Each log category has its own level. Higher levels capture more detail but produce larger logs (max 20MB -- truncated after that).

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
# tail logs in real time
sf apex tail log --target-org <your-org-alias>

# tail with FINEST level on Apex Code
sf apex tail log --target-org <your-org-alias> --debug-level FINEST

# run anonymous Apex and capture output
sf apex run --file scripts/apex/myScript.apex --target-org <your-org-alias>

# list recent debug logs
sf apex list log --target-org <your-org-alias>

# get a specific log by ID
sf apex get log --log-id <logId> --target-org <your-org-alias>

# get the most recent log
sf apex get log --number 1 --target-org <your-org-alias>
```

To set a debug log for a specific user in Setup: Setup > Debug Logs > New. Logs are retained for 24 hours.

---

## 3. Reading a Debug Log

Every log line follows this structure:

```
15:42:07.123 (1234567890)|EVENT_TYPE|[line number]|detail message
```

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

To find where an exception originated: search for `EXCEPTION_THROWN`, then look at `CODE_UNIT_STARTED` lines above it to reconstruct the call stack.

---

## 4. Limit Tracking Pattern

Add these during debugging to see how many governor limit slots a code path consumes. Remove before any production deploy.

```apex
System.debug('SOQL used: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
System.debug('DML used: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());
System.debug('Heap used: ' + Limits.getHeapSize() + '/' + Limits.getLimitHeapSize() + ' bytes');
System.debug('CPU used: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime() + ' ms');
System.debug('Callouts used: ' + Limits.getCallouts() + '/' + Limits.getLimitCallouts());
```

Utility method pattern during development:

```apex
// dev-only -- delete before deploy
private static void logLimits(String checkpoint) {
    System.debug('LIMITS @ ' + checkpoint +
        ' | SOQL: ' + Limits.getQueries() + '/' + Limits.getLimitQueries() +
        ' | DML: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements() +
        ' | CPU: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime() + 'ms');
}
```

Never ship `Limits.*` calls -- they add log noise in production.

---

## 5. Common Errors Reference

| Error Message | Root Cause | Fix |
|---|---|---|
| `System.LimitException: Too many SOQL queries: 101` | SOQL inside a for loop | Move query before the loop, use a Map keyed by Id |
| `System.NullPointerException: Attempt to de-reference a null object` | Accessing `.field` on a null SObject | Add null check before field access |
| `FIELD_INTEGRITY_EXCEPTION` | Lookup field set to wrong-type record Id | Verify the record exists and Id type matches |
| `MIXED_DML_OPERATION` | Setup object and non-setup object in same transaction | Move setup DML to `@future` or use `System.runAs()` in tests |
| `UNABLE_TO_LOCK_ROW` | Two transactions updating the same record | Reduce transaction scope, add retry logic, or use Platform Events |
| `Callout from scheduled Apex not supported` | `System.schedule()` used in a test for a callout-enabled Schedulable | Test via `new MySchedulable().execute(null)` directly |
| `Entity is deleted` | Code accessed a soft-deleted record | Query `ALL ROWS` if needed, or handle with try/catch |
| `List has no rows for assignment to SObject` | SOQL assigned directly to a single SObject, returned 0 rows | Use `List<SObject>` and check `isEmpty()` |
| `Too many email invocations: 11` | `Messaging.sendEmail()` called inside a loop | Collect into a list, call `sendEmail()` once |
| `Apex CPU time limit exceeded` | Heavy synchronous processing | Offload to `Queueable` or `Batch` Apex |
| `SObject row retrieved via SOQL without querying the requested field` | Field not in SELECT clause | Add the field to your SOQL |
| `You cannot deploy to a required field` | A `required=true` custom field is in `fieldPermissions` in a Permission Set | Remove that field from `fieldPermissions` |
| `Read only: field is not writeable` | Attempting to set a formula or auto-number field | Remove the field from the SObject assignment |

---

## 6. Anonymous Apex for Quick Tests

Use anonymous Apex to test a single method without deploying a test class:

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

Steps via Setup:
1. Setup > Debug Logs > New
2. Select "User" as the traced entity type
3. Pick the specific user reporting the issue
4. Set log levels to the minimum needed (DEBUG for Apex Code, WARN for everything else)
5. Reproduce the issue as that user
6. Retrieve the log within 24 hours

```bash
# tail logs in real time while user reproduces
sf apex tail log --target-org <your-org-alias>

# retrieve a specific log by ID
sf apex get log --log-id 07Lxx000000xxxx --target-org <your-org-alias> --output-dir logs/
```

Log retention: 24 hours from creation OR until 250 MB org log storage is full.

Production debug checklist:
- Enable for the specific user only
- Set minimum log levels needed
- Reproduce immediately after enabling
- Download the log before 24h expires
- Disable the trace flag when done

---

## Sources

- [Debug Log Documentation](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_debugging_debug_log.htm)
- [Developer Console Overview](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/code_dev_console.htm)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [System.Limits Class Reference](https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_methods_system_limits.htm)
