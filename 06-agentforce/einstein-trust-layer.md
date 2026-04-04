# Einstein Trust Layer

**Your org's data never goes to the LLM directly. The Trust Layer sits in between. Here's what it actually does.**

---

## What It Does

- Masks PII fields before sending to the model. The LLM sees `[MASKED]` instead of the actual value.
- Never stores your data outside Salesforce. Prompts and responses aren't used for model training.
- Logs every LLM call with a full audit trail: who called it, what was sent (with masking applied), what came back, when.
- Filters toxic or harmful output before it reaches the user.
- Validates that the agent stays within its configured topic scope.

---

## What It Doesn't Do

It's not a full DLP solution. A few things it won't protect against:

- Apex actions can access any data the running user has permission to see. If your action queries and returns Social Security Numbers, the Trust Layer masks them before the LLM sees them, but your Apex code still processed the raw value. The Apex action is your responsibility.
- It doesn't prevent prompt injection from adversarial user input. Handle that through topic scoping and out-of-scope rules in your system prompt.
- It doesn't restrict which Salesforce records the agent can read. That's the sharing model and FLS, enforced in your Apex actions.

---

## Checking the Audit Trail

Setup > Einstein > Trust Layer > Audit Trail.

Every call shows:
- The masked prompt sent to the LLM
- The model response
- Which fields were masked and what rule applied
- The user who initiated the conversation
- Timestamp and session ID

Use this to debug unexpected agent responses and to verify masking is working.

---

## Masking Configuration

Fields masked by default:

| Field Type | Default Behaviour |
|---|---|
| Email | Masked |
| Phone | Masked |
| SSN (pattern-matched) | Masked |
| Credit card numbers (pattern-matched) | Masked |
| Standard fields: `PersonEmail`, `MobilePhone`, `SSN__c` | Masked if field type matches |

To add custom masking rules: Setup > Einstein > Trust Layer > Data Masking > New Rule.

You can mask by field API name, field type, or regex pattern on the value.

---

## Implication for Agentforce Actions

Your Apex action returns a result. That result passes to the Trust Layer. The Trust Layer masks PII. Then the LLM uses the masked result to generate its response.

The LLM sees the masked version. But the summary string you return from Apex is what the LLM uses most directly. If your summary contains `"Customer email: john@example.com"`, the Trust Layer masks the email in the prompt, but the LLM might still infer context from surrounding text.

Structure your action output to avoid putting sensitive values in the summary.

**Wrong:**
```apex
res.summary = 'Found contact ' + contact.Name + ' with email ' + contact.Email;
```

**Right:**
```apex
res.summary = 'Found an active contact record for ' + contact.Name + '. 1 open case assigned.';
```

If you wouldn't say it out loud to a customer, don't put it in the summary.
