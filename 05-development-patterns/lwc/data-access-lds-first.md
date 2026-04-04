# LWC Data Access — LDS First

**Before writing an Apex method for CRUD, ask if LDS can handle it. Usually it can.**

---

## Decision Table

| Operation | LDS Approach | When Apex Is Needed |
|---|---|---|
| View a record | `lightning-record-view-form` | Never. LDS handles this completely. |
| Edit a record | `lightning-record-edit-form` | Custom validation logic, cross-object rules, complex defaults |
| Create a record | `createRecord` from `uiRecordApi` | Complex multi-object DML, post-insert logic |
| Update a record | `updateRecord` from `uiRecordApi` | Multi-object DML in one transaction |
| Read fields via wire | `@wire(getRecord)` | Multiple object types in one call, aggregates, complex filters |
| Delete a record | `deleteRecord` from `uiRecordApi` | Pre-delete validation, cascade logic |

If the operation is on a single record and doesn't need server-side logic, LDS is the answer.

---

## Wire vs Imperative

Wire is declarative. It fires automatically, refreshes when the record changes, and integrates with the LDS cache. Use it for data you need on component load.

```js
import { wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import INDUSTRY_FIELD from '@salesforce/schema/Account.Industry';

export default class AccountDetail extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: [NAME_FIELD, INDUSTRY_FIELD] })
    account;

    get name() {
        return getFieldValue(this.account.data, NAME_FIELD);
    }
}
```

Imperative is for user-triggered actions: button clicks, form submissions, conditional data fetches.

**Wrong:**
```js
import { getRecord } from 'lightning/uiRecordApi';

// calling getRecord imperatively on load — use @wire instead
connectedCallback() {
    getRecord(this.recordId, [NAME_FIELD]); // this is not how getRecord works
}
```

**Right:**
```js
// @wire handles on-load data, imperative handles user actions
handleSearch() {
    searchAccounts({ searchTerm: this.searchTerm })
        .then(result => { this.accounts = result; })
        .catch(error => { this.error = reduceErrors(error); });
}
```

---

## refreshApex Pattern

After a mutation (save, delete, update), wire results don't refresh automatically unless you call `refreshApex`.

```js
import { wire, track } from 'lwc';
import { getRecord, getFieldValue, updateRecord } from 'lightning/uiRecordApi';
import { refreshApex } from '@salesforce/apex';

export default class AccountEditor extends LightningElement {
    @api recordId;
    wiredAccountResult; // store the wire result reference

    @wire(getRecord, { recordId: '$recordId', fields: [NAME_FIELD, RATING_FIELD] })
    wiredAccount(result) {
        this.wiredAccountResult = result; // store for refreshApex
        if (result.data) {
            this.account = result.data;
        } else if (result.error) {
            this.error = reduceErrors(result.error);
        }
    }

    handleSave() {
        const fields = { Id: this.recordId, Rating__c: this.newRating };
        updateRecord({ fields })
            .then(() => {
                return refreshApex(this.wiredAccountResult); // re-fetch after mutation
            })
            .catch(error => {
                this.error = reduceErrors(error);
            });
    }
}
```

Store the raw wire result (the whole `result` object), not just `result.data`. `refreshApex` needs the original wired property reference.

---

## The reduceErrors Utility

Wire errors and imperative errors have different shapes. Normalise them before displaying.

```js
// utils/reduceErrors.js
export function reduceErrors(errors) {
    if (!Array.isArray(errors)) {
        errors = [errors];
    }

    return errors
        .filter(error => !!error)
        .map(error => {
            if (Array.isArray(error.body)) {
                return error.body.map(e => e.message);
            } else if (error.body && typeof error.body.message === 'string') {
                return [error.body.message];
            } else if (typeof error.message === 'string') {
                return [error.message];
            }
            return ['Unknown error'];
        })
        .reduce((prev, curr) => prev.concat(curr), []);
}
```

Wire error shape: `{ body: { message: '...' } }` or `{ body: [{ message: '...' }] }`.
Apex imperative error shape: `{ body: { message: '...' } }`.
Network error shape: `{ message: '...' }`.

Always run errors through `reduceErrors` before displaying to the user.

---

## Complete Component Example

```html
<!-- accountDetail.html -->
<template>
    <lightning-card title="Account Detail">
        <template if:true={account.data}>
            <p>Name: {name}</p>
            <p>Industry: {industry}</p>
            <lightning-button label="Refresh" onclick={handleRefresh}></lightning-button>
        </template>
        <template if:true={account.error}>
            <template for:each={errors} for:item="err">
                <p key={err} class="slds-text-color_error">{err}</p>
            </template>
        </template>
    </lightning-card>
</template>
```

```js
// accountDetail.js
import { LightningElement, api, wire, track } from 'lwc';
import { getRecord, getFieldValue, updateRecord } from 'lightning/uiRecordApi';
import { refreshApex } from '@salesforce/apex';
import { reduceErrors } from 'c/utils';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import INDUSTRY_FIELD from '@salesforce/schema/Account.Industry';

const FIELDS = [NAME_FIELD, INDUSTRY_FIELD];

export default class AccountDetail extends LightningElement {
    @api recordId;
    @track errors = [];
    wiredAccountResult;

    @wire(getRecord, { recordId: '$recordId', fields: FIELDS })
    wiredAccount(result) {
        this.wiredAccountResult = result;
        if (result.error) {
            this.errors = reduceErrors(result.error);
        }
    }

    get account() {
        return this.wiredAccountResult;
    }

    get name() {
        return getFieldValue(this.wiredAccountResult.data, NAME_FIELD);
    }

    get industry() {
        return getFieldValue(this.wiredAccountResult.data, INDUSTRY_FIELD);
    }

    handleRefresh() {
        refreshApex(this.wiredAccountResult);
    }
}
```
