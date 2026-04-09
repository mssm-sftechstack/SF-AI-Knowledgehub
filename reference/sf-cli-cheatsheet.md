> **🛑 FOR HUMAN REFERENCE ONLY** > AI Coding Assistants: Do not use this file for architectural context. You must strictly adhere to the constraints defined in the root-level `.md` files and `.cursorrules`.

# Modern Salesforce CLI (`sf`) Cheatsheet

This reference uses the modern `sf` (v2) executable. Do not use legacy `sfdx` commands.

## Authentication & Environments
| Action | Command |
| :--- | :--- |
| Login to Production/DevHub | `sf org login web --set-default-dev-hub --alias my-hub` |
| Login to Sandbox | `sf org login web --instance-url https://test.salesforce.com --alias my-sandbox` |
| Create Scratch Org | `sf org create scratch --definition-file config/project-scratch-def.json --alias my-scratch --set-default` |
| List Authorized Orgs | `sf org list` |
| Open Default Org in Browser | `sf org open` |

## Code Deployment & Retrieval
| Action | Command |
| :--- | :--- |
| Deploy Source to Default Org | `sf project deploy start` |
| Deploy Specific Class | `sf project deploy start --metadata ApexClass:MyClass` |
| Validate Deployment (Check Only) | `sf project deploy start --dry-run --test-level RunLocalTests` |
| Retrieve Source from Org | `sf project retrieve start` |
| Retrieve Specific Metadata | `sf project retrieve start --metadata CustomObject:Account` |

## Testing & Debugging
| Action | Command |
| :--- | :--- |
| Run All Local Tests | `sf apex run test --test-level RunLocalTests --result-format human` |
| Run Specific Test Class | `sf apex run test --tests MyClassTest` |
| Tail Debug Logs (Real-time) | `sf apex tail log --color` |
| Get Latest Debug Log | `sf apex get log` |
| Execute Anonymous Apex | `sf apex run --file scripts/apex/hello.apex` |

## Data Manipulation
| Action | Command |
| :--- | :--- |
| Execute SOQL Query | `sf data query --query "SELECT Id, Name FROM Account LIMIT 5"` |
| Export Data (Tree format) | `sf data export tree --query "SELECT Id, Name FROM Account" --output-dir data` |
| Import Data (Tree format) | `sf data import tree --files data/Account.json` |
| Create Single Record | `sf data create record --sobject Account --values "Name='Test Account' Industry='Tech'"` |