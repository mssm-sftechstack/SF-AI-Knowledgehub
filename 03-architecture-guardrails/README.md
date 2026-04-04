# Architecture Guardrails

**The patterns you need to know before your AI tool writes a single line of Salesforce code.**

Salesforce is a shared platform. The rules aren't suggestions. Breaking them works fine in dev, then blows up at scale in production in front of customers. These pages cover the architecture decisions that matter most.

## Contents

| Page | What it covers |
|---|---|
| [Multi-Tenant Fundamentals](multitenant-fundamentals.md) | Governor limits, the shared resource model, and why "it works in dev" is a trap |
| [Apex Layered Architecture](apex-layered-architecture.md) | Trigger → Handler → Service → Selector pattern with real code examples |
| [Order of Execution](order-of-execution.md) | The 17-step sequence every Salesforce developer needs to know |
| [Async Patterns](async-decision-tree.md) | @future vs Queueable vs Batch vs Schedulable, with a decision table |
| [SOQL Performance](soql-performance.md) | Selectivity, indexes, and the anti-patterns that cause CPU limit exceptions |
| [Agentforce Architecture](agentforce-architecture.md) | How LLMs, Topics, Actions, and the Einstein Trust Layer connect |
