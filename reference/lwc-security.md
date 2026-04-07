# LWC Security

XSS prevention, sanitization, and secure DOM patterns.

---

## Rule: Never Use innerHTML with User Input

### Wrong (XSS)

```javascript
export default class UserBio extends LightningElement {
  @track bio;
  
  connectedCallback() {
    getUserBio().then(result => {
      this.bio = result;  // User input from server
    });
  }
}

<!-- Template -->
<div innerHTML={bio}></div>
```

If `bio` = `<img onerror="alert('xss')">`, the alert runs. XSS vulnerability.

### Right (Text Binding)

```html
<!-- Template -->
<div>{bio}</div>  <!-- Automatically escaped -->
```

LWC escapes HTML special characters automatically. Safe.

### Right (Sanitized HTML)

If you truly need HTML (rich text):

```javascript
import DOMPurify from 'isomorphic-dompurify';

export default class UserBio extends LightningElement {
  @track safeBio;
  
  connectedCallback() {
    getUserBio().then(result => {
      this.safeBio = DOMPurify.sanitize(result);  // Remove dangerous tags
    });
  }
}

<!-- Template -->
<div innerHTML={safeBio}></div>
```

DOMPurify removes `<script>`, `<img onerror>`, and other attack vectors.

---

## Rule: Validate Input

### Wrong (Trusts User Input)

```javascript
async handleSave() {
  const name = this.template.querySelector('input').value;
  await saveAccount({ name });  // What if name = 'Rob\'; DROP TABLE Account; --'?
}
```

If `name` is used in unescaped SOQL, SQL injection (though Salesforce SOQL is safe by default with bind variables).

### Right (Validate)

```javascript
async handleSave() {
  const name = this.template.querySelector('input').value;
  
  // Validate
  if (!name || name.trim().length === 0) {
    this.error = 'Name is required';
    return;
  }
  
  if (name.length > 255) {
    this.error = 'Name too long';
    return;
  }
  
  await saveAccount({ name });
}
```

---

## Rule: Escape URLs

### Wrong (Unsafe)

```html
<a href={userUrl}>Click here</a>

<!-- If userUrl = 'javascript:alert(1)', it executes -->
```

### Right (Safe URLs)

```javascript
export default class Link extends LightningElement {
  get safeUrl() {
    // Only allow http/https URLs
    if (this.userUrl && (this.userUrl.startsWith('http://') || this.userUrl.startsWith('https://'))) {
      return this.userUrl;
    }
    return '#';  // Default to # if invalid
  }
}

<!-- Template -->
<a href={safeUrl}>Click here</a>
```

---

## Rule: Use SLDS for UI

SLDS (Salesforce Lightning Design System) has security baked in.

### Wrong (Custom HTML)

```html
<div style="color: red; background: blue;">Error</div>
```

Risks: Inline style injection, accessibility issues.

### Right (SLDS)

```html
<lightning-card title="Form">
  <div class="slds-form-element">
    <label class="slds-form-element__label" for="input-name">Name</label>
    <div class="slds-form-element__control">
      <input type="text" id="input-name" class="slds-input" />
    </div>
  </div>
</lightning-card>
```

SLDS handles styling, accessibility, and security.

---

## Rule: Don't Expose Sensitive Data

### Wrong (Logs PII)

```javascript
async handleError(error) {
  console.error('User email:', user.email, 'Error:', error);  // PII in console
}
```

If console is visible to others, PII exposed.

### Right (Safe Logging)

```javascript
async handleError(error) {
  console.error('Error processing user:', this.userId, 'Error code:', error.body?.errorCode);
  // Log ID, not email
  
  // Send to Apex for secure logging
  logError({ userId: this.userId, errorCode: error.body?.errorCode });
}
```

---

## Rule: Sanitize Rich Text

### Wrong

```javascript
this.innerHTML = userContent;  // User could paste <script>
```

### Right (First Approach: Plain Text)

```html
{userContent}  <!-- Automatically escaped -->
```

### Right (Second Approach: Sanitized HTML)

```javascript
import DOMPurify from 'isomorphic-dompurify';

sanitizedContent = DOMPurify.sanitize(userContent, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],  // Only allow safe tags
  ALLOWED_ATTR: ['href']
});

// Template
<div innerHTML={sanitizedContent}></div>
```

---

## Rule: Validate Apex Responses

### Wrong (Trusts Apex)

```javascript
async handleFetch() {
  const result = await getUser();
  // result = { email: 'user@example.com', ssn: '123-45-6789' }
  this.user = result;  // Displays all data
}

<!-- Template -->
<p>{user.email}</p>
<p>{user.ssn}</p>
```

If Apex returns sensitive data, it's exposed in UI.

### Right (Validate in Apex)

```apex
// Apex
@AuraEnabled
public static User getUserPublic() {
  User u = [SELECT Id, Email FROM User WHERE Id = :UserInfo.getUserId()];
  // Only return safe fields
  return u;
}
```

OR

### Right (Filter in Component)

```javascript
async handleFetch() {
  const result = await getUser();
  // Only use what's needed
  this.user = {
    email: result.email
    // Don't copy ssn
  };
}
```

---

## CSP Trusted Sites

If your LWC calls external APIs:

Setup > Security > CSP Trusted Sites

```
Trusted Site Name: Google_API
Trusted Site URL: https://www.googleapis.com
CSP Directive: connect-src
```

This allows LWC to fetch from Google API.

---

## Checklist: LWC Security Review

- ✅ No innerHTML with user input (or sanitized)
- ✅ Input validation (length, format)
- ✅ URLs validated (http/https only)
- ✅ No PII in logs
- ✅ SLDS used for UI
- ✅ Apex methods check CRUD/FLS
- ✅ Error messages don't expose sensitive info
- ✅ No XSS vectors (test with malicious input)
- ✅ CSP Trusted Sites configured if needed
- ✅ Keyboard navigation working

