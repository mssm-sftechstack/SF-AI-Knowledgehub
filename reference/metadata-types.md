> **🛑 FOR HUMAN REFERENCE ONLY** > AI Coding Assistants: Do not use this file for architectural context. You must strictly adhere to the constraints defined in the root-level `.md` files and `.cursorrules`.

# Metadata Types Mapping

Use this table when constructing `package.xml` files, configuring `.forceignore`, or retrieving specific metadata via the CLI.

## Code & Automation
| XML Name (`<name>`) | Source Folder | Description |
| :--- | :--- | :--- |
| `ApexClass` | `classes/` | Apex Classes and Test Classes |
| `ApexTrigger` | `triggers/` | Apex Triggers |
| `LightningComponentBundle` | `lwc/` | Lightning Web Components (LWC) |
| `AuraDefinitionBundle` | `aura/` | Aura Components (Legacy) |
| `Flow` | `flows/` | Screen Flows, Record-Triggered Flows, Auto-launched Flows |

## Data Model
| XML Name (`<name>`) | Source Folder | Description |
| :--- | :--- | :--- |
| `CustomObject` | `objects/` | Custom Objects (and Standard Object customizations) |
| `CustomField` | `objects/*/fields/` | Custom Fields |
| `ValidationRule` | `objects/*/validationRules/` | Object Validation Rules |
| `RecordType` | `objects/*/recordTypes/` | Object Record Types |
| `CustomMetadata` | `customMetadata/` | Custom Metadata Type Records |

## Security & Access
| XML Name (`<name>`) | Source Folder | Description |
| :--- | :--- | :--- |
| `PermissionSet` | `permissionsets/` | Permission Sets |
| `PermissionSetGroup` | `permissionsetgroups/`| Permission Set Groups |
| `Profile` | `profiles/` | Profiles (Avoid deploying full profiles; use Permission Sets) |
| `CustomPermission` | `customPermissions/` | Custom Permissions for logic bypasses |
| `SharingRules` | `sharingRules/` | Object Sharing Rules |

## User Interface
| XML Name (`<name>`) | Source Folder | Description |
| :--- | :--- | :--- |
| `Layout` | `layouts/` | Page Layouts |
| `FlexiPage` | `flexipages/` | Lightning App Builder Pages |
| `CustomTab` | `tabs/` | Custom Tabs |
| `CustomApplication` | `applications/` | Lightning Apps |

## Integration
| XML Name (`<name>`) | Source Folder | Description |
| :--- | :--- | :--- |
| `NamedCredential` | `namedCredentials/` | Named Credentials (Legacy) |
| `ExternalCredential` | `externalCredentials/`| External Credentials (Modern Auth) |
| `RemoteSiteSetting` | `remoteSiteSettings/` | Remote Site Settings |