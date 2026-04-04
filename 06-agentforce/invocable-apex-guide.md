# Invocable Apex for Agentforce Actions

**The description on `@InvocableMethod` is what the AI reads to decide when to call your action. Write it precisely.**

---

## The Annotations

```apex
public with sharing class CaseAgentActions {

    @InvocableMethod(
        label='Get Open Cases for Account'
        description='Returns open support cases for a given account. Use when the user asks about support history, unresolved issues, or active tickets for an account.'
        category='Case Management'
    )
    public static List<CaseResponse> getOpenCases(List<CaseRequest> requests) {
        // implementation
    }
}
```

`@InvocableVariable` on the inner classes:

```apex
public class CaseRequest {
    @InvocableVariable(label='Account ID' description='Salesforce ID of the account' required=true)
    public Id accountId;
}

public class CaseResponse {
    @InvocableVariable(label='Cases' description='List of open cases for the account')
    public List<Case> cases;

    @InvocableVariable(label='Summary' description='Human-readable summary for the agent to present to the user')
    public String summary;

    @InvocableVariable(label='Case Count' description='Total number of open cases')
    public Integer caseCount;
}
```

---

## The Description Rule

The `description` on `@InvocableMethod` is what the LLM reads to decide which action to invoke. Be specific and action-oriented.

| Bad | Good |
|---|---|
| "Gets account data" | "Returns the account's name, industry, and annual revenue. Use when the user asks what the account does or requests basic account information." |
| "Updates a case" | "Escalates a case to high priority and assigns it to Tier 2 support. Use when the user says the issue is urgent, blocking, or impacting production." |
| "Checks cases" | "Returns open support cases for a given account. Use when the user asks about support history, unresolved issues, or active tickets." |

The LLM picks actions based on semantic similarity between the user's message and your description. Vague descriptions cause the wrong action to fire, or no action at all.

---

## The Summary Pattern

Return a `summary` String in your output class. That's the text the agent uses to respond to the user. Without it, the LLM has to interpret raw SObject data, which produces inconsistent output.

```apex
public static List<CaseResponse> getOpenCases(List<CaseRequest> requests) {
    List<CaseResponse> responses = new List<CaseResponse>();

    for (CaseRequest req : requests) {
        List<Case> cases = [
            SELECT Id, Subject, Status, Priority, CreatedDate
            FROM Case
            WHERE AccountId = :req.accountId
            AND Status != 'Closed'
            WITH SECURITY_ENFORCED
            ORDER BY CreatedDate DESC
        ];

        CaseResponse res = new CaseResponse();
        res.cases = cases;
        res.caseCount = cases.size();

        if (cases.isEmpty()) {
            res.summary = 'This account has no open support cases.';
        } else {
            res.summary = 'Found ' + cases.size() + ' open case(s). '
                + 'Most recent: "' + cases[0].Subject + '" (Status: ' + cases[0].Status + ', Priority: ' + cases[0].Priority + ').';
        }

        responses.add(res);
    }

    return responses;
}
```

Keep the summary short (1-3 sentences). Don't echo PII or sensitive field values in the summary.

---

## Idempotency

Agent actions can be called multiple times if the conversation is retried or the agent re-evaluates. Design for it.

**Wrong:**
```apex
// inserts on every call, creates duplicates if retried
public static List<CaseResponse> createOnboardingCase(List<CaseRequest> requests) {
    insert new Case(Subject = 'Onboarding', AccountId = requests[0].accountId);
    // ...
}
```

**Right:**
```apex
// check-then-act, safe on retry
public static List<CaseResponse> createOnboardingCase(List<CaseRequest> requests) {
    Id accountId = requests[0].accountId;
    List<Case> existing = [
        SELECT Id FROM Case
        WHERE AccountId = :accountId AND Subject = 'Onboarding' AND Status != 'Closed'
        WITH SECURITY_ENFORCED
        LIMIT 1
    ];

    if (!existing.isEmpty()) {
        CaseResponse res = new CaseResponse();
        res.summary = 'An onboarding case already exists for this account.';
        return new List<CaseResponse>{ res };
    }

    insert new Case(Subject = 'Onboarding', AccountId = accountId, Status = 'New');
    CaseResponse res = new CaseResponse();
    res.summary = 'Onboarding case created successfully.';
    return new List<CaseResponse>{ res };
}
```

---

## Always with sharing

Agent actions run in the context of the logged-in user. Use `with sharing` so the sharing model applies. If the user can't see a record, the action shouldn't return it.

```apex
public with sharing class CaseAgentActions {
    // user can only see what they have access to
}
```

---

## Complete Working Example

```apex
public with sharing class CaseAgentActions {

    public class CaseRequest {
        @InvocableVariable(label='Account ID' description='Salesforce Account ID' required=true)
        public Id accountId;

        @InvocableVariable(label='Status Filter' description='Filter cases by status (optional). Leave blank for all open cases.')
        public String statusFilter;
    }

    public class CaseResponse {
        @InvocableVariable(label='Case Count')
        public Integer caseCount;

        @InvocableVariable(label='Summary')
        public String summary;

        @InvocableVariable(label='Has Open Cases')
        public Boolean hasOpenCases;
    }

    @InvocableMethod(
        label='Get Open Cases for Account'
        description='Returns open support cases for a given account. Use when the user asks about support history, unresolved issues, or active tickets.'
        category='Case Management'
    )
    public static List<CaseResponse> getOpenCases(List<CaseRequest> requests) {
        List<CaseResponse> responses = new List<CaseResponse>();

        for (CaseRequest req : requests) {
            String statusClause = String.isNotBlank(req.statusFilter) ? req.statusFilter : 'Open';

            List<Case> cases = [
                SELECT Id, Subject, Status, Priority, CaseNumber
                FROM Case
                WHERE AccountId = :req.accountId
                AND Status != 'Closed'
                WITH SECURITY_ENFORCED
                ORDER BY CreatedDate DESC
                LIMIT 50
            ];

            CaseResponse res = new CaseResponse();
            res.caseCount = cases.size();
            res.hasOpenCases = !cases.isEmpty();

            if (cases.isEmpty()) {
                res.summary = 'No open cases found for this account.';
            } else {
                res.summary = cases.size() + ' open case(s) found. '
                    + 'Most recent: Case ' + cases[0].CaseNumber + ' - "' + cases[0].Subject
                    + '" is ' + cases[0].Priority + ' priority.';
            }

            responses.add(res);
        }

        return responses;
    }
}
```

---

## Testing Invocable Methods

Wrap the input in a List, call the method, assert the summary string.

```apex
@IsTest
static void testGetOpenCases_withOpenCases() {
    Account acc = TestDataFactory.createAccount(true);
    insert new Case(Subject = 'Login broken', Status = 'New', Priority = 'High', AccountId = acc.Id);

    CaseAgentActions.CaseRequest req = new CaseAgentActions.CaseRequest();
    req.accountId = acc.Id;

    Test.startTest();
    List<CaseAgentActions.CaseResponse> responses =
        CaseAgentActions.getOpenCases(new List<CaseAgentActions.CaseRequest>{ req });
    Test.stopTest();

    System.assertEquals(1, responses.size(), 'One response expected');
    System.assertEquals(1, responses[0].caseCount, 'One open case expected');
    System.assert(responses[0].summary.contains('1 open case'), 'Summary should mention count');
    System.assertEquals(true, responses[0].hasOpenCases, 'hasOpenCases should be true');
}

@IsTest
static void testGetOpenCases_noOpenCases() {
    Account acc = TestDataFactory.createAccount(true);

    CaseAgentActions.CaseRequest req = new CaseAgentActions.CaseRequest();
    req.accountId = acc.Id;

    Test.startTest();
    List<CaseAgentActions.CaseResponse> responses =
        CaseAgentActions.getOpenCases(new List<CaseAgentActions.CaseRequest>{ req });
    Test.stopTest();

    System.assertEquals(0, responses[0].caseCount, 'No cases expected');
    System.assertEquals(false, responses[0].hasOpenCases, 'hasOpenCases should be false');
}
```
