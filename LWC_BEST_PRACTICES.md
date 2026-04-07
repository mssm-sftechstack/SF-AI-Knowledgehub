# LWC Best Practices

Lightning Web Components (LWC) are modern JavaScript components for Salesforce UIs. This guide covers lifecycle, state management, error handling, accessibility, and testing.

---

## Lifecycle Hooks

LWC has four main lifecycle hooks that fire in order:

### 1. constructor()

Runs once when component is created. Use for initialization.

```javascript
export default class MyComponent extends LightningElement {
  counter = 0;
  
  constructor() {
    super();
    console.log('Component created, counter =', this.counter);
  }
}
```

**Use for**: Setting default values, initializing state.

**Don't use for**: DOM access, Apex calls (too early, DOM not ready).

### 2. connectedCallback()

Runs when component is added to DOM. Use for setup that needs DOM.

```javascript
export default class MyComponent extends LightningElement {
  connectedCallback() {
    console.log('Component in DOM');
    this.loadData();
  }
  
  loadData() {
    // Safe to use this.template.querySelector() here
  }
}
```

**Use for**: Apex calls, loading initial data, subscribing to events.

**Don't use for**: Final DOM access (use renderedCallback).

### 3. renderedCallback()

Runs after every render cycle (after HTML updates). Use for DOM access.

```javascript
export default class MyComponent extends LightningElement {
  @track isLoading = false;
  
  renderedCallback() {
    // DOM is now fully updated
    const button = this.template.querySelector('[data-id="submit"]');
    if (button) {
      button.focus();  // Set focus after render
    }
  }
}
```

**Use for**: DOM manipulation, focus management, scrolling, animations.

**Don't use for**: Apex calls in a loop (causes performance issues).

### 4. disconnectedCallback()

Runs when component is removed from DOM. Use for cleanup.

```javascript
export default class MyComponent extends LightningElement {
  unsubscribeHandler;
  
  connectedCallback() {
    // Subscribe to event
    this.unsubscribeHandler = subscribe(null, MY_CHANNEL, (message) => {
      this.handleMessage(message);
    });
  }
  
  disconnectedCallback() {
    // Unsubscribe (prevent memory leak)
    unsubscribe(this.unsubscribeHandler);
  }
}
```

**Use for**: Cleaning up subscriptions, timers, event listeners.

---

## State Management with @track

The `@track` decorator tells LWC to track a property for reactivity.

### Primitive Values

```javascript
export default class MyComponent extends LightningElement {
  @track firstName = 'John';
  
  handleChange(event) {
    this.firstName = event.target.value;  // Triggers re-render
  }
}
```

### Objects & Arrays

```javascript
export default class MyComponent extends LightningElement {
  @track user = {
    name: 'John',
    email: 'john@example.com'
  };
  
  @track items = [];
  
  addItem(item) {
    // ❌ Wrong (doesn't trigger re-render)
    this.items.push(item);
    
    // ✅ Right (creates new array, triggers re-render)
    this.items = [...this.items, item];
  }
  
  updateUser() {
    // ❌ Wrong (mutating object, may not trigger re-render)
    this.user.name = 'Jane';
    
    // ✅ Right (creates new object, triggers re-render)
    this.user = { ...this.user, name: 'Jane' };
  }
}
```

**Key Rule**: For arrays and objects, create a new instance to trigger re-rendering. Mutation alone may not work.

---

## Wire Adapters for Data

Wire adapters fetch data reactively. Use `@wire` decorator.

### Wiring Apex Methods

```javascript
import { LightningElement, wire } from 'lwc';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

export default class AccountList extends LightningElement {
  accounts;
  error;
  
  @wire(getAccounts)
  wiredAccounts({ data, error }) {
    if (data) {
      this.accounts = data;
      this.error = undefined;
    } else if (error) {
      this.error = error;
      this.accounts = undefined;
    }
  }
}
```

### Wiring with Parameters

```javascript
export default class AccountDetail extends LightningElement {
  @track accountId;
  account;
  error;
  
  @wire(getAccount, { accountId: '$accountId' })
  wiredAccount({ data, error }) {
    if (data) {
      this.account = data;
    } else if (error) {
      this.error = error;
    }
  }
}
```

The `$` prefix makes it reactive: when `accountId` changes, wire automatically re-fetches.

### Wiring Record Data (LDS)

```javascript
import { LightningElement, wire } from 'lwc';
import { getRecord } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import PHONE_FIELD from '@salesforce/schema/Account.Phone';

export default class AccountDetail extends LightningElement {
  @track recordId = '0011q00000YeKrQAAV';
  
  @wire(getRecord, { 
    recordId: '$recordId', 
    fields: [NAME_FIELD, PHONE_FIELD]
  })
  account;
  
  get accountName() {
    return getFieldValue(this.account.data, NAME_FIELD);
  }
  
  get accountPhone() {
    return getFieldValue(this.account.data, PHONE_FIELD);
  }
}
```

---

## Error Handling

### Pattern 1: Try-Catch for Apex Calls

```javascript
export default class MyComponent extends LightningElement {
  @track isLoading = false;
  error = null;
  
  async handleSave() {
    try {
      this.isLoading = true;
      this.error = null;
      
      const result = await createAccount({ name: 'Acme' });
      console.log('Account created:', result);
      
      // Show success message
      this.dispatchEvent(
        new ShowToastEvent({
          title: 'Success',
          message: 'Account created',
          variant: 'success'
        })
      );
    } catch (error) {
      console.error('Error creating account:', error);
      this.error = error.body.message;
      
      this.dispatchEvent(
        new ShowToastEvent({
          title: 'Error',
          message: this.error,
          variant: 'error'
        })
      );
    } finally {
      this.isLoading = false;
    }
  }
}
```

### Pattern 2: Handling Wire Errors

```javascript
export default class AccountList extends LightningElement {
  accounts;
  error;
  
  @wire(getAccounts)
  wiredAccounts({ data, error }) {
    if (data) {
      this.accounts = data;
      this.error = undefined;
    } else if (error) {
      this.error = 'Unable to load accounts: ' + error.body.message;
      this.accounts = undefined;
    }
  }
  
  get hasError() {
    return this.error !== undefined;
  }
}
```

### Pattern 3: User-Friendly Error Messages

```javascript
handleError(error) {
  let message = 'An error occurred';
  
  if (error.body) {
    if (error.body.message) {
      message = error.body.message;  // Server error
    } else if (Array.isArray(error.body)) {
      message = error.body[0].message;  // Multiple errors, take first
    }
  } else {
    message = error.message;  // Network error
  }
  
  this.dispatchEvent(
    new ShowToastEvent({
      title: 'Error',
      message: message,
      variant: 'error'
    })
  );
}
```

---

## Accessibility (a11y)

### Rule 1: Use Semantic HTML

```html
<!-- ❌ Wrong (div is not accessible) -->
<div onclick={handleClick} role="button">Click me</div>

<!-- ✅ Right (button is semantic) -->
<button onclick={handleClick}>Click me</button>
```

### Rule 2: Label Form Inputs

```html
<!-- ❌ Wrong (no label) -->
<input type="text" placeholder="Name">

<!-- ✅ Right (input has label) -->
<label for="nameInput">Name</label>
<input id="nameInput" type="text">
```

### Rule 3: Alt Text for Images

```html
<!-- ❌ Wrong (no alt text) -->
<img src="icon.png">

<!-- ✅ Right (alt text provided) -->
<img src="icon.png" alt="Account icon">

<!-- ✅ Right (decorative, alt empty) -->
<img src="decoration.png" alt="">
```

### Rule 4: Color is Not the Only Indicator

```html
<!-- ❌ Wrong (only red color indicates error) -->
<span style="color: red;">Error</span>

<!-- ✅ Right (text and color) -->
<span class="error-icon">⚠️</span> <span>Error message</span>
```

### Rule 5: Keyboard Navigation

```javascript
export default class Modal extends LightningElement {
  handleKeyDown(event) {
    if (event.key === 'Escape') {
      this.closeModal();  // Close on Escape
    }
  }
}
```

---

## DOM Access Best Practices

### Pattern 1: Using this.template

```javascript
export default class MyComponent extends LightningElement {
  handleClick() {
    // ✅ Right (scoped to component)
    const button = this.template.querySelector('button');
    
    // ❌ Wrong (searches entire document)
    const button = document.querySelector('button');
  }
}
```

### Pattern 2: Safe DOM Queries

```javascript
renderedCallback() {
  const input = this.template.querySelector('[data-id="email"]');
  
  if (input) {  // Always check if element exists
    input.focus();
  }
}
```

### Pattern 3: Avoiding Common Mistakes

```javascript
// ❌ Wrong (mutating DOM directly)
connectedCallback() {
  const div = this.template.querySelector('div');
  div.innerHTML = '<strong>Bold</strong>';  // XSS risk
}

// ✅ Right (using LWC binding)
template: `
  <template if:true={isBold}>
    <strong>{text}</strong>
  </template>
`
```

---

## Conditional Rendering

### Using if:true and if:false

```html
<template if:true={isLoading}>
  <div class="spinner">Loading...</div>
</template>

<template if:false={isLoading}>
  <table>
    <template for:each={accounts} for:item="account">
      <tr key={account.id}>
        <td>{account.name}</td>
      </tr>
    </template>
  </table>
</template>
```

### Using if:true with @track

```javascript
export default class MyComponent extends LightningElement {
  @track showDetails = false;
  
  toggleDetails() {
    this.showDetails = !this.showDetails;
  }
}
```

---

## Looping Best Practices

### Use key for Unique Identity

```html
<template for:each={items} for:item="item">
  <div key={item.id} data-id={item.id}>
    {item.name}
  </div>
</template>
```

The `key` attribute must be unique for each item in the list. Without it, LWC can't track which item changed.

### Avoid Complex Expressions in Loops

```html
<!-- ❌ Wrong (calling method in loop) -->
<template for:each={items} for:item="item">
  <p>{getFormattedDate(item.date)}</p>
</template>

<!-- ✅ Right (pre-process in JavaScript) -->
<!-- Template: -->
<template for:each={formattedItems} for:item="item">
  <p>{item.formattedDate}</p>
</template>

<!-- JavaScript: -->
get formattedItems() {
  return this.items.map(item => ({
    ...item,
    formattedDate: this.formatDate(item.date)
  }));
}
```

---

## Common Mistakes

### Mistake 1: Modifying Array Without Re-render

```javascript
// ❌ Wrong
this.items.push(newItem);  // Array changes, but re-render may not happen

// ✅ Right
this.items = [...this.items, newItem];  // New array, triggers re-render
```

### Mistake 2: Missing $ in Wire Parameters

```javascript
// ❌ Wrong (accountId doesn't react to changes)
@wire(getAccount, { accountId: accountId })

// ✅ Right (accountId reacts to changes)
@wire(getAccount, { accountId: '$accountId' })
```

### Mistake 3: XSS Vulnerability (innerHTML)

```javascript
// ❌ Wrong
this.message = '<strong>Bold</strong>';
template: `<div>{message}</div>`  // Safe (automatic escaping)
this.innerHTML = message;  // XSS risk if message contains <img onerror>

// ✅ Right
template: `<div>{message}</div>`  // Auto-escaped, safe
```

### Mistake 4: Not Handling Wire Errors

```javascript
// ❌ Wrong (no error handling)
@wire(getAccounts)
wiredAccounts(data) {
  this.accounts = data;
}

// ✅ Right (handles both data and error)
@wire(getAccounts)
wiredAccounts({ data, error }) {
  if (data) {
    this.accounts = data;
  } else if (error) {
    this.error = error;
  }
}
```

### Mistake 5: Not Unsubscribing from Channels

```javascript
// ❌ Wrong (memory leak)
connectedCallback() {
  subscribe(null, MY_CHANNEL, this.handleMessage);
}

// ✅ Right (clean up on disconnect)
unsubscribeHandler;

connectedCallback() {
  this.unsubscribeHandler = subscribe(null, MY_CHANNEL, this.handleMessage);
}

disconnectedCallback() {
  unsubscribe(this.unsubscribeHandler);
}
```

---

## Testing LWC with Jest

See LWC Jest testing guide for detailed test patterns.

---

## Checklist: LWC Ready for Production

- ✅ Uses `@wire` for data loading (reactive)
- ✅ Error handling for all Apex calls
- ✅ `@track` for reactive state
- ✅ No hardcoded IDs
- ✅ Accessibility: labels, alt text, semantic HTML
- ✅ Keyboard navigation working
- ✅ No XSS vulnerabilities (no innerHTML)
- ✅ DOM access after renderedCallback
- ✅ Memory cleanup in disconnectedCallback
- ✅ Loops have unique `key` attributes
- ✅ All wire calls handle error case
- ✅ Responsive design (mobile-friendly)

