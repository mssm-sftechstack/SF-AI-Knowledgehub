# Real-World AI Failures (The Wall of Shame)

AI models confidently write code that looks correct but fails in Salesforce production environments. This log documents real-world AI failures and how our repository constraints fix them.

## Failure 1: The LWC Wire Adapter Memory Leak
**The Prompt:** "Write an LWC to display Account details and update the status when a button is clicked."

**The AI Hallucination:**
The AI uses `@wire` to fetch the record, but then attempts to mutate the wired property directly when the button is clicked. 
* **The Crash:** Wired properties in LWC are immutable. Attempting to change `this.account.data.Status = 'Active'` throws a read-only error in the browser and crashes the component.

**The Architect Fix:**
Enforce that wired data must be shallow-copied before mutation.
```javascript
// ✅ Correct LWC Mutation Pattern
let clonedAccount = { ...this.account.data };
clonedAccount.Status = 'Active';
```

## Failure 2: The Future Method Collision
**The Prompt:** "Make an external callout when an Account is updated."

**The AI Hallucination:**
The AI writes a `@future(callout=true)` method and calls it from the `AccountTrigger`.
* **The Crash:** If the Account update was initiated by a Batch Apex job, calling a `@future` method from inside a Batch job throws a `System.AsyncException: Future method cannot be called from a future or batch method`.

**The Architect Fix:**
Enforce System context checks before calling asynchronous code, or use Queueable Apex.
```apex
// ✅ Correct Asynchronous Dispatch Pattern
if (!System.isFuture() && !System.isBatch()) {
    System.enqueueJob(new AccountCalloutQueueable(accountIds));
}
```

## Failure 3: The Dynamic SOQL Injection
**The Prompt:** "Write a method to search for Contacts by a user-provided Last Name."

**The AI Hallucination:**
The AI concatenates the search string directly into the database query.
* **The Crash:** A malicious user types `Smith' OR Name != '` into the search bar, bypassing the filter entirely and returning the entire database.

**The Architect Fix:**
Enforce the use of bind variables for all dynamic inputs.
```apex
// ✅ Correct Dynamic Search Pattern
public List<Contact> searchContacts(String searchTerm) {
    String query = 'SELECT Id, Name FROM Contact WHERE LastName LIKE :searchTerm WITH USER_MODE';
    return Database.query(query);
}
```

## Failure 4: The Cross-Object DML Trap
**The Prompt:** "When an Opportunity is Won, update the parent Account Revenue, then update all child Quotes."

**The AI Hallucination:**
The AI writes the logic in an `after update` trigger, performs DML on the Account, and then performs a separate DML on the Quotes.
* **The Crash:** If the Account DML triggers an Account Flow, which in turn updates the Opportunity, the Opportunity trigger fires again, creating a recursive loop that destroys the CPU limits.

**The Architect Fix:**
Enforce static recursion checks in Trigger Handlers.
```apex
// ✅ Correct Recursion Control
public class OpportunityTriggerHandler {
    public static Boolean isFirstTime = true;
    
    public static void afterUpdate(List<Opportunity> opps) {
        if (!isFirstTime) return;
        isFirstTime = false;
        // Proceed with complex cross-object DML
    }
}
```