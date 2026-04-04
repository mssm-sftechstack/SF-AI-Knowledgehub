---
name: sf-lwc
description: >
  Salesforce LWC development skill. Covers HTML templates, JS controllers, CSS,
  meta XML, wire adapters, Lightning Data Service, lifecycle hooks, accessibility,
  SLDS, error handling, security (Lightning Web Security), and Agentforce Actions
  integration. Based on LWC Developer Guide (API 62.0).
allowed-tools: Bash(sf project deploy*)
---

# Salesforce LWC Skill

**HTML templates, JS controllers, wire adapters, Lightning Data Service, accessibility, and Lightning Web Security.**

## Rule 1 — LWC for All New Development
Use Aura ONLY for: Utility Bar, Overlay Library, Lightning Out. Everything else = LWC.

**Reference:** [LWC vs Aura](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_migrate_introduction.htm)

## Rule 2 — Lightning Data Service First
Before writing Apex for CRUD, ask: can LDS handle this?

| Operation | LDS Component | When to Use Apex Instead |
|---|---|---|
| View a record | `lightning-record-view-form` | Never |
| Edit a record | `lightning-record-edit-form` | When custom validation/logic needed |
| Full form with layout | `lightning-record-form` | When custom layout needed |
| Read via wire | `@wire(getRecord)` | When multiple objects needed |
| Create/update via JS | `createRecord`, `updateRecord` | When complex DML needed |

```javascript
// prefer LDS over direct Apex for standard CRUD
import { getRecord, updateRecord } from 'lightning/uiRecordApi';
import ACCOUNT_NAME from '@salesforce/schema/Account.Name';

@wire(getRecord, { recordId: '$recordId', fields: [ACCOUNT_NAME] })
wiredAccount({ data, error }) {
    if (data) this.accountName = data.fields.Name.value;
    else if (error) this.error = error;
}
```

**Reference:** [Lightning Data Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_data_ui_api.htm)

---

## Required File Structure
```
force-app/main/default/lwc/
└── componentName/
    ├── componentName.html
    ├── componentName.js
    ├── componentName.js-meta.xml
    └── componentName.css        (optional)
```

---

## HTML Template Rules
```html
<template>
    <lightning-card title="Accounts" icon-name="standard:account">

        <!-- loading -->
        <template lwc:if={isLoading}>
            <lightning-spinner alternative-text="Loading" size="small"></lightning-spinner>
        </template>

        <!-- error -->
        <template lwc:if={error}>
            <div class="slds-notify slds-notify_alert slds-theme_error" role="alert">
                <span class="slds-assistive-text">Error</span>
                {errorMessage}
            </div>
        </template>

        <!-- data -->
        <template lwc:if={accounts}>
            <ul class="slds-list_dotted">
                <template for:each={accounts} for:item="account">
                    <li key={account.Id} class="slds-item">
                        <lightning-button
                            label={account.Name}
                            onclick={handleAccountClick}
                            data-id={account.Id}
                            aria-label={account.Name}>
                        </lightning-button>
                    </li>
                </template>
            </ul>
        </template>

    </lightning-card>
</template>
```

**Never:**
- Hardcode user-facing strings — use `@salesforce/label`
- Use inline styles — use SLDS utility classes
- Skip `key` on iterated elements
- Skip `aria-label` on interactive elements
- Use `innerHTML` — XSS risk

---

## JavaScript Controller Patterns
```javascript
import { LightningElement, wire, api, track } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { refreshApex } from '@salesforce/apex';
import getAccounts from '@salesforce/apex/AccountSelector.getAccounts';
import LABEL_ERROR from '@salesforce/label/c.Error_Title';
import ACCOUNT_NAME from '@salesforce/schema/Account.Name';

export default class AccountList extends NavigationMixin(LightningElement) {

    @api recordId;
    @api cardTitle = 'Accounts'; // always provide defaults for @api props

    // @track is only needed for object/array mutation detection
    // primitives (string, boolean, number) are reactive without @track since Spring '20
    @track filter = { status: 'Active', type: 'Customer' };

    error;
    isLoading = false;
    _wiredResult; // store wire result for refreshApex

    labels = { errorTitle: LABEL_ERROR };

    // wire is declarative and auto-refreshes; use imperative only for user-triggered actions
    @wire(getAccounts, { status: '$filter.status' })
    wiredAccounts(result) {
        this._wiredResult = result; // store for refreshApex
        const { data, error } = result;
        if (data) {
            this.accounts = data;
            this.error = undefined;
        } else if (error) {
            this.error = error;
            this.accounts = undefined;
        }
    }

    // connectedCallback runs after the component inserts into the DOM
    connectedCallback() {
        // good place to set up non-DOM event listeners or start polling
    }

    // disconnectedCallback: always clean up subscriptions, listeners, timers here
    disconnectedCallback() {
        // unsubscribe from message channels, clear intervals, remove event listeners
        if (this._messageSubscription) {
            unsubscribe(this._messageSubscription);
        }
    }

    // imperative Apex call, only for user-triggered actions
    async handleRefresh() {
        this.isLoading = true;
        try {
            await refreshApex(this._wiredResult); // refreshes the wire cache
        } catch (e) {
            this.error = e;
            this.toast('error', this.labels.errorTitle, this.reduceErrors(e));
        } finally {
            this.isLoading = false;
        }
    }

    handleAccountClick(event) {
        const accountId = event.currentTarget.dataset.id;
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: { recordId: accountId, objectApiName: 'Account', actionName: 'view' }
        });
    }

    // wire and imperative errors have different shapes — always normalise
    reduceErrors(errors) {
        if (!Array.isArray(errors)) errors = [errors];
        return errors
            .filter(e => !!e)
            .map(e => e.body?.message || e.message || String(e))
            .join(', ');
    }

    get errorMessage() {
        return this.reduceErrors(this.error);
    }

    toast(variant, title, message) {
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
    }
}
```

---

## Lightning Message Service (Cross-Component Communication)
Use LMS instead of application events for communication between unrelated components.
```javascript
// publisher
import { publish, MessageContext } from 'lightning/messageService';
import WEATHER_CHANNEL from '@salesforce/messageChannel/WeatherUpdate__c';

@wire(MessageContext) messageContext;

publishUpdate(data) {
    publish(this.messageContext, WEATHER_CHANNEL, { weatherData: data });
}

// subscriber
import { subscribe, unsubscribe, MessageContext } from 'lightning/messageService';

@wire(MessageContext) messageContext;
_subscription;

connectedCallback() {
    this._subscription = subscribe(this.messageContext, WEATHER_CHANNEL, (msg) => {
        this.handleWeatherUpdate(msg.weatherData);
    });
}

disconnectedCallback() {
    unsubscribe(this._subscription); // always unsubscribe
}
```

**Reference:** [Lightning Message Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_use_message_channel.htm)

---

## Lightning Web Security (LWS)

Lightning Web Security is the security architecture that replaced Lightning Locker Service for LWC components.

| Aspect | Locker Service (older) | Lightning Web Security (current) |
|---|---|---|
| DOM isolation | Shadow DOM + Locker wrappers | Shadow DOM + LWS per-component isolation |
| Third-party libraries | Many blocked by Locker | Broader support (jQuery, Moment.js work) |
| Cross-component DOM access | Blocked | Still blocked — components cannot reach other components' DOM |
| `window` and `document` | Sandboxed | Sandboxed, but more compatible |
| Applies to | Aura + LWC in older orgs | LWC in current orgs (Aura still uses Locker) |

### What Still Applies Under LWS
- `innerHTML`, `outerHTML`, `insertAdjacentHTML` — still XSS risks, still blocked by policy
- `eval()` and `new Function()` — still disallowed
- Cross-component DOM queries (`document.querySelector` across shadow boundaries) — still blocked
- Third-party libraries that manipulate the global DOM — may work differently

### How to Check Which Security Model Your Org Uses
```
Setup → Session Settings → Security → "Use Lightning Web Security for Lightning web components"
```

```javascript
// safe under both Locker and LWS — use these patterns
this.template.querySelector('.my-class');         // within this component's shadow
this.template.querySelectorAll('lightning-input'); // within this component

// unsafe under both — reaches across shadow boundary
document.querySelector('.other-components-class'); // WRONG
```

**Reference:** [Lightning Web Security Developer Guide](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_security_lwsec.htm)

---

## Agentforce Actions — Invocable Apex from AI Agents

Expose Apex methods as Agentforce Actions so AI agents can invoke business logic:

```apex
public with sharing class WeatherAgentActions {

    // InvocableMethod makes this callable by Agentforce, Flow, and Process Builder
    @InvocableMethod(
        label='Get Current Weather'
        description='Returns current weather data for a given city. Use when the user asks about weather conditions.'
        category='Weather'
    )
    public static List<WeatherResult> getWeather(List<WeatherRequest> requests) {
        List<WeatherResult> results = new List<WeatherResult>();
        for (WeatherRequest req : requests) {
            WeatherData data = WeatherService.fetchWeather(req.cityName);
            WeatherResult r = new WeatherResult();
            r.temperature  = data.temperature;
            r.humidity     = data.humidity;
            r.summary      = data.cityName + ': ' + data.temperature + '°C, humidity ' + data.humidity + '%';
            results.add(r);
        }
        return results;
    }

    public class WeatherRequest {
        @InvocableVariable(label='City Name' description='The city to get weather for' required=true)
        public String cityName;
    }

    public class WeatherResult {
        @InvocableVariable(label='Temperature')
        public Decimal temperature;

        @InvocableVariable(label='Humidity')
        public Integer humidity;

        @InvocableVariable(label='Summary')
        public String summary;  // AI agent uses this as the response to the user
    }
}
```

### Agent Action Design Rules
- Keep `description` on `@InvocableMethod` precise — the AI uses it to decide when to call this action
- Keep `description` on each `@InvocableVariable` accurate — the AI uses it to map user intent to parameters
- Actions must be idempotent and handle null/empty inputs gracefully
- Return a human-readable `summary` string — agents surface this directly to the user
- Keep action scope narrow — one action per business function, not a Swiss-army-knife method
- Test the action in Agentforce Builder before deploying

**Reference:** [Agentforce Actions](https://developer.salesforce.com/docs/einstein/genai/guide/actions-overview.html)

---

## Meta XML
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordPage</target>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <property name="cardTitle" type="String" label="Card Title" default="My Component"/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

---

## Security Rules
- NEVER use `innerHTML`, `outerHTML`, `insertAdjacentHTML` — XSS vulnerability
- NEVER use `eval()` or `Function()` with external data
- NEVER store credentials or tokens in component state
- NEVER log sensitive field values to console
- Always use `@salesforce/schema` field imports for type safety
- Sanitize user input before passing to Apex
- Use Shadow DOM boundaries correctly — do not reach across component boundaries

**Reference:** [LWC Security Best Practices](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_security_xss.htm)

---

## Deployment (Confirm with User First)
```bash
sf project deploy start --metadata LightningComponentBundle:componentName --target-org <your-org-alias>
```

---

## LWC Quality Scoring (Ship only at 132+/165)

Score every LWC component before marking it ready. Minimum threshold: 132/165 (80%).

| Dimension | Points | Failing Condition |
|---|---|---|
| Security | 25 | innerHTML with user data; eval(); credentials in component state; no XSS protection |
| Accessibility | 20 | Interactive elements without aria-label; no keyboard navigation; colour as only state indicator |
| Performance | 20 | Wire adapter not used for reactive data; no loading state; @track on primitive; no lazy loading for large lists |
| Architecture | 20 | Component does 5+ unrelated things; parent reaches into child DOM; no LDS for standard CRUD |
| LDS Usage | 20 | Apex used where LDS handles it (view/edit single record); no refreshApex on mutation; LDS cache not invalidated |
| Error Handling | 20 | No error state in template; raw JSON error shown to user; async without try/catch; no finally for isLoading |
| Code Quality | 20 | Hardcoded user-facing strings (no @salesforce/label); inline styles; missing key on iteration; console.log left in |
| Testing (Jest) | 20 | No Jest tests; no wire mock; no DOM assertion; no event test |
| **TOTAL** | **165** | |

**Score 132-165** → ready to ship
**Score 110-131** → fix failing dimensions first
**Score below 110** → reject, do not deploy

### PICKLES Architecture — Reference Checklist
Use PICKLES to verify a complete LWC component is production-ready:

- **P**rototype — does the component render correctly with mock data?
- **I**ntegrate — does it wire/callout correctly to real data?
- **C**omposition — is it split into parent (layout) + child (atomic) where needed?
- **K**inetics — do all user interactions (click, input, navigation) work?
- **L**ibraries — are all imports accounted for (no unused imports)?
- **E**rror — does every async path have a loading state + error state?
- **S**ecurity — aria labels, no innerHTML, field schema imports, no PII in console

---

## Official Sources
- [LWC Developer Guide](https://developer.salesforce.com/docs/component-library/documentation/en/lwc)
- [Lightning Web Components Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.lwc.meta/lwc/lwc_intro.htm)
- [Lightning Data Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_data_ui_api.htm)
- [Lightning Message Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_use_message_channel.htm)
- [Lightning Web Security](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_security_lwsec.htm)
- [Agentforce Actions](https://developer.salesforce.com/docs/einstein/genai/guide/actions-overview.html)
- [LWC Security Best Practices](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc_security_xss.htm)
- [Well-Architected: Easy (Intentional)](https://architect.salesforce.com/well-architected/easy/intentional)
- [Well-Architected: Secure](https://architect.salesforce.com/well-architected/trusted/secure)
