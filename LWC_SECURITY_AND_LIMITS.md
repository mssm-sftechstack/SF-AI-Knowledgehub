# Enterprise LWC Security & Performance Constraints

**CRITICAL DIRECTIVE FOR AI:** Lightning Web Components (LWC) run entirely on the client side. You must treat the client as a hostile, easily manipulated environment. Do not generate LWC code that compromises data security, violates the Shadow DOM, or creates infinite rendering loops.

## 1. The Data Leakage Trap (The Cacheable Fallacy)
AI frequently suggests `@AuraEnabled(cacheable=true)` for all Apex methods to improve performance. This is dangerous. 

* **The Rule:** NEVER use `cacheable=true` on methods returning Personally Identifiable Information (PII), Protected Health Information (PHI), or highly dynamic financial data. Caching stores this data in the browser cache, creating a severe data leak vulnerability.
* **Imperative vs. Wired:** Use Wire adapters ONLY for read-only, non-sensitive data. Use Imperative Apex for DML operations and sensitive data retrievals.

```mermaid
graph TD
    A[Component Mounts] --> B{Is Data Sensitive or Requires DML?}
    B -->|Yes| C[Imperative Apex Call]
    B -->|No| D[Wire Adapter]
    D --> E{Is Data Highly Dynamic?}
    E -->|Yes| C
    E -->|No| F[@AuraEnabled cacheable=true]
    
    style C fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style F fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

## 2. Shadow DOM Enforcement
Salesforce enforces Lightning Web Security (LWS) and Locker Service.
* NEVER generate code using `document.getElementById()` or `document.querySelector()`. 
* ALWAYS use `this.template.querySelector()` to interact with the DOM. You cannot pierce the Shadow DOM to manipulate parent or sibling components.

## 3. The `renderedCallback` Infinite Loop
AI frequently attempts to update reactive properties (`@track` or primitive properties) inside `renderedCallback()`. 
* **The Rule:** NEVER update a reactive property inside `renderedCallback()` without a strict boolean guard (`isRendered`). Doing so will trigger another render cycle, resulting in a browser-crashing infinite loop.

```javascript
// ✅ MANDATORY RENDEREDCALLBACK GUARD PATTERN
renderedCallback() {
    if (this.hasRendered) {
        return;
    }
    this.hasRendered = true;
    
    // Safe to perform one-time DOM manipulations here
}
```