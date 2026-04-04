# Trigger Framework

**One trigger per object. Delegate everything to a handler. Keep the trigger file under 15 lines.**

---

## Anti-Pattern

```apex
trigger AccountTrigger on Account (before insert, before update, after insert, after update, after delete) {
    if (Trigger.isBefore && Trigger.isInsert) {
        for (Account acc : Trigger.new) {
            if (acc.Industry == null) {
                acc.Industry = 'Technology';
            }
            // 50 more lines of logic...
        }
    }
    if (Trigger.isBefore && Trigger.isUpdate) {
        // another 80 lines...
    }
    if (Trigger.isAfter && Trigger.isInsert) {
        List<Task> tasks = new List<Task>();
        for (Account acc : Trigger.new) {
            // DML inside a conditional block, mixed with routing logic
        }
        insert tasks;
    }
}
```

300-line triggers with nested if/else blocks. Logic duplicated across trigger files. No way to bypass without a deployment.

---

## The Pattern

Three files. The trigger stays dumb. The handler routes. Services do the work.

### AccountTrigger.trigger

```apex
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    new AccountTriggerHandler().run();
}
```

That's it. Under 15 lines. Always.

### AccountTriggerHandler.cls

```apex
public with sharing class AccountTriggerHandler extends TriggerHandler {

    private List<Account> newList;
    private List<Account> oldList;
    private Map<Id, Account> newMap;
    private Map<Id, Account> oldMap;

    public AccountTriggerHandler() {
        this.newList = (List<Account>) Trigger.new;
        this.oldList = (List<Account>) Trigger.old;
        this.newMap  = (Map<Id, Account>) Trigger.newMap;
        this.oldMap  = (Map<Id, Account>) Trigger.oldMap;
    }

    protected override void beforeInsert() {
        AccountService.setDefaultIndustry(newList);
    }

    protected override void afterInsert() {
        AccountService.createOnboardingTasks(newList);
    }

    protected override void beforeUpdate() {
        AccountService.validateRatingChange(newList, oldMap);
    }
}
```

### TriggerHandler.cls (base class)

```apex
public virtual class TriggerHandler {

    private static Set<String> bypassedHandlers = new Set<String>();

    public void run() {
        if (isBypassed()) return;

        if (Trigger.isBefore) {
            if (Trigger.isInsert)   beforeInsert();
            if (Trigger.isUpdate)   beforeUpdate();
            if (Trigger.isDelete)   beforeDelete();
        } else {
            if (Trigger.isInsert)   afterInsert();
            if (Trigger.isUpdate)   afterUpdate();
            if (Trigger.isDelete)   afterDelete();
            if (Trigger.isUndelete) afterUndelete();
        }
    }

    // subclasses override only what they need
    protected virtual void beforeInsert()   {}
    protected virtual void beforeUpdate()   {}
    protected virtual void beforeDelete()   {}
    protected virtual void afterInsert()    {}
    protected virtual void afterUpdate()    {}
    protected virtual void afterDelete()    {}
    protected virtual void afterUndelete()  {}

    private Boolean isBypassed() {
        String handlerName = String.valueOf(this).split(':')[0];
        return bypassedHandlers.contains(handlerName)
            || FeatureManagement.checkPermission('Bypass_Account_Trigger');
    }

    public static void bypass(String handlerName) {
        bypassedHandlers.add(handlerName);
    }

    public static void clearBypass(String handlerName) {
        bypassedHandlers.remove(handlerName);
    }
}
```

---

## The Bypass Pattern

Don't use static booleans. They don't survive async context boundaries.

**Wrong:**
```apex
// static variable dies at @future / Queueable boundary
public class AccountTriggerHandler {
    public static Boolean bypassTrigger = false;
}
// In Queueable.execute(): AccountTriggerHandler.bypassTrigger resets to false
```

Use Custom Permissions instead. They travel with the running user, not the stack frame.

```apex
// In TriggerHandler.isBypassed()
FeatureManagement.checkPermission('Bypass_Account_Trigger')
```

Assign the Custom Permission to a permission set. Give it to integration users, data migration profiles, or admins running bulk jobs. Revoke it when you're done.

---

## When to Add a Method vs When a New Class Is Warranted

| Situation | Decision |
|---|---|
| New field validation on Account | Add to `beforeUpdate()` in `AccountTriggerHandler` |
| Complex account scoring with 5+ methods | Extract to `AccountScoringService`, call from handler |
| Account changes trigger Opportunity updates | `OpportunityService` called from `AccountTriggerHandler.afterUpdate()` |
| Second trigger needed on Account | Never. Extend the handler. |
| Logic applies to multiple objects | Extract to a shared service, call from each handler |

The handler's job is routing. Keep each handler method to 2-5 lines. If a handler method hits 30 lines, extract a service.
