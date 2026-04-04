# Persona and System Prompt Design

**The system prompt is the agent's personality and rulebook combined. Vague prompts produce unpredictable agents.**

---

## What Belongs in a System Prompt

Four things, in this order:

1. **Who the agent is.** Name, role, tone.
2. **What it handles.** Specific scope. The topics it can help with.
3. **What it never does.** Explicit out-of-scope list.
4. **Escalation triggers.** Exactly when and how to hand off to a human.

---

## Length Rule

Stay under 2000 characters. LLMs don't process long instructions better. They process them worse. Beyond a certain length, instructions at the top of the prompt get deprioritised in favour of recent conversation context.

If you need more than 2000 characters to describe the agent, it's trying to do too much. Split it into multiple agents with separate topics.

---

## The Escalation Rule

Always define exactly when the agent hands off. Without this, the agent keeps trying to answer questions it can't handle.

```
Escalate to a human agent when:
- The user says "agent", "human", "representative", or "supervisor"
- The user expresses frustration a second time in the same session
- The request involves a refund over $500
- The user describes a safety or legal issue
- You cannot resolve the issue after two attempts
```

Be specific about the trigger. "Escalate when appropriate" is not a rule.

---

## Out-of-Scope Blocking

Explicitly list what the agent won't do. Otherwise it improvises.

```
This agent does not:
- Access or modify billing or payment information
- Make promises about shipping dates or refunds
- Provide legal or medical advice
- Discuss competitor products
- Retrieve or share account credentials
```

If you don't say it won't, the LLM will try. It'll either hallucinate an answer or attempt an action it doesn't have.

---

## PII Rule

The agent should never echo PII back to the user. The Trust Layer masks PII before it goes to the model, but your summary strings in Apex actions are what the agent actually says. Don't include email addresses, phone numbers, SSNs, or payment card data in summary strings.

**Wrong:**
```apex
res.summary = 'Found account for ' + contact.Email + '. Their phone is ' + contact.Phone;
```

**Right:**
```apex
res.summary = 'Found an active account for ' + contact.Name + '. Case ' + caseNumber + ' is open.';
```

---

## Good vs Bad System Prompt

**Bad:**

```
You are a helpful Salesforce support agent. Help users with their questions about accounts, cases,
and orders. Be polite and professional. If you can't help, let them know. Always try to resolve
the customer's issue. You can look up information and create records as needed. If something is
too complex, escalate to a human. Don't share sensitive information. Be concise.
```

Problems: scope is undefined, escalation trigger is vague, "sensitive information" is undefined, "as needed" gives unbounded permission to create records.

**Good:**

```
You are Aria, a support assistant for Acme Corp. You help customers with order status,
returns, and product questions.

You can: look up orders, check return eligibility, provide product specs, create return
requests for eligible orders.

You cannot: process payments, modify pricing, access another customer's account, make
commitments not listed in our return policy.

Escalate to a human when:
- The user says "agent", "human", or "supervisor"
- The user expresses frustration twice
- The issue involves a payment dispute or legal claim
- You've attempted to resolve the issue twice and failed

Never include email addresses, phone numbers, or payment details in your responses.
Keep responses under 3 sentences unless the user asks for more detail.
```

Clear scope. Specific permissions. Explicit out-of-scope list. Defined escalation triggers. PII rule. Response length guidance. Under 1500 characters.
