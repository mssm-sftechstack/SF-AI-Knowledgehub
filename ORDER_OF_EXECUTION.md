# Order of Execution: The AI Blindspot

**CRITICAL DIRECTIVE:** AI models view code linearly. Salesforce executes code in a complex, multi-layered pipeline. If an AI writes an After Trigger without knowing a Record-Triggered Flow runs later in the transaction, it will cause recursion and crash the org.

## The Execution Pipeline
You must evaluate this pipeline before generating any Apex Trigger or Flow logic. 

```mermaid
graph TD
    A[Record Insert/Update Triggered] --> B(System Validations)
    B --> C{Before-Save Flows}
    C --> D[Before Apex Triggers]
    D --> E(Custom Validation Rules)
    E --> F[(Save to Database - Not Committed)]
    F --> G[After Apex Triggers]
    G --> H[Assignment & Auto-Response Rules]
    H --> I[Workflow Rules]
    I --> J{Record-Triggered Flows - After Save}
    J --> K[Roll-up Summary Fields Calculated]
    K --> L[(Commit to Database)]
    
    style C fill:#f9f0ff,stroke:#a371f7,stroke-width:2px
    style J fill:#f9f0ff,stroke:#a371f7,stroke-width:2px
    style F fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px
    style L fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

## Architectural Mandates for AI Generation
1. **Never assume an empty transaction.** Always assume other automations (Flows, Managed Packages) are running in the same context.
2. **Prioritize Before-Save Flows.** For same-record field updates, always recommend a Before-Save Flow over a Before-Insert/Update Apex Trigger.
3. **Beware of After-Save Recursion.** If writing an After Trigger that updates related records, you must implement a recursion check to prevent cross-object infinite loops.