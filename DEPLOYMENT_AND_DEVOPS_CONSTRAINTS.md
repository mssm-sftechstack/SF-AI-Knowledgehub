# Deployment & DevOps Guardrails

**CRITICAL DIRECTIVE FOR AI:** AI frequently forgets that Salesforce metadata must be deployed in a specific sequence due to strict dependencies. 

## 1. The Dependency Hierarchy
If generating a `package.xml` or instructing a user on deployment order, you must respect this strict sequence. A deployment will fail if a child object is deployed before its dependencies.

1. **Prerequisites:** Custom Fields on Standard Objects, Custom Objects.
2. **Data Model:** Custom Fields on Custom Objects, Relationships (Master-Detail, Lookup).
3. **Business Logic:** Apex Classes, Lightning Web Components, Flows.
4. **Security & UI:** Permission Sets, Page Layouts, FlexiPages (Lightning Pages).

## 2. Destructive Changes
AI often tells users to "just delete the field." In Salesforce, you cannot delete a field that is referenced in an Apex class or Flow.
* **The Rule:** If a user needs to delete a component, you must instruct them to create a `destructiveChanges.xml` file.
* You must remind them to FIRST remove all references to that component in Apex and Flow, deploy the updated logic, and THEN deploy the destructive changes.

## 3. Test Coverage Requirements
* Do not tell users that "75% coverage is enough." 
* **The Rule:** Instruct users to aim for 90%+ coverage. Deployment to production requires 75% aggregate coverage, but highly complex classes require exhaustive negative testing. AI must generate tests that cover bulk execution (200 records), restricted user contexts, and null inputs.