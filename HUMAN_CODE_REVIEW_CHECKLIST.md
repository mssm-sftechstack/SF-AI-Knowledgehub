# The Human Code Review Checklist

**CRITICAL DIRECTIVE:** AI models are confident liars. Even with strict system prompts, they will occasionally hallucinate a bypass to governor limits or security models. It is the primary responsibility of the human developer to audit the generated code before deployment.

Do not approve an AI-generated Pull Request until you have verified the following:

## 1. The Multi-Tenant Sanity Check
- [ ] **No loops contain SOQL or DML.** Scan every `for` and `while` loop. Are there any `[SELECT...]` or `insert/update` statements inside?
- [ ] **Bulkification is proven.** Does the method accept a `List<SObject>` or `Set<Id>`? If it accepts a single `Id`, reject it.
- [ ] **Collections are used efficiently.** Are `Map<Id, SObject>` and `Set<Id>` used to relate data rather than nested loops?

## 2. The Zero-Trust Security Check
- [ ] **Sharing is explicit.** Does the Apex class have the `with sharing` keyword?
- [ ] **Queries are restricted.** Do all SOQL queries (API 55.0+) end with `WITH USER_MODE`?
- [ ] **DML is restricted.** Do all DML operations use the `as user` keyword or `Security.stripInaccessible()`?
- [ ] **No Dynamic Injection.** If the AI wrote dynamic SOQL (`Database.query()`), is it strictly using bind variables (`:myVar`) instead of string concatenation?

## 3. The Architecture & Flow Check
- [ ] **Apex wasn't used to bypass Flow.** Could this logic have been accomplished faster and safer with a Before-Save Fast Field Update Flow?
- [ ] **Recursion is handled.** If this is an After-Trigger, is there a static boolean or set-based check to prevent it from firing infinitely?
- [ ] **No synchronous callouts in triggers.** If there is an `Http.send()`, is it safely isolated inside an `@future` or `Queueable` class?

## 4. The Test Authenticity Check (Anti-Lazy AI)
- [ ] **No SeeAllData.** Is `@isTest(SeeAllData=true)` completely absent?
- [ ] **Real Assertions.** Does the test class contain `Assert.areEqual()` verifying the *actual data state change*, or did the AI just call the method to get 75% line coverage?
- [ ] **Bulk Testing.** Does the test factory insert at least 200 records to prove the code survives governor limits?