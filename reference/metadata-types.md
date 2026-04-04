# Metadata Types Reference

**The metadata API names, file extensions, and folder paths you'll use most.**

---

## Apex

| Type | Metadata API Name | File Extension | Folder |
|---|---|---|---|
| Apex Class | `ApexClass` | `.cls` + `.cls-meta.xml` | `force-app/main/default/classes/` |
| Apex Trigger | `ApexTrigger` | `.trigger` + `.trigger-meta.xml` | `force-app/main/default/triggers/` |
| Apex Test Class | `ApexClass` | `.cls` + `.cls-meta.xml` | Same as classes |

---

## LWC and Aura

| Type | Metadata API Name | Folder |
|---|---|---|
| Lightning Web Component | `LightningComponentBundle` | `force-app/main/default/lwc/<componentName>/` |
| Aura Component | `AuraDefinitionBundle` | `force-app/main/default/aura/<componentName>/` |

Each LWC folder contains: `.html`, `.js`, `.js-meta.xml`, optionally `.css` and `.svg`.

---

## Objects and Fields

| Type | Metadata API Name | File Extension | Folder |
|---|---|---|---|
| Custom Object | `CustomObject` | `.object-meta.xml` | `force-app/main/default/objects/<ObjectName__c>/` |
| Custom Field | `CustomField` | `.field-meta.xml` | `force-app/main/default/objects/<ObjectName__c>/fields/` |
| Record Type | `RecordType` | `.recordType-meta.xml` | `force-app/main/default/objects/<ObjectName__c>/recordTypes/` |
| Validation Rule | `ValidationRule` | `.validationRule-meta.xml` | `force-app/main/default/objects/<ObjectName__c>/validationRules/` |

---

## Permissions and Security

| Type | Metadata API Name | File Extension | Folder |
|---|---|---|---|
| Permission Set | `PermissionSet` | `.permissionset-meta.xml` | `force-app/main/default/permissionsets/` |
| Custom Permission | `CustomPermission` | `.customPermission-meta.xml` | `force-app/main/default/customPermissions/` |
| Profile | `Profile` | `.profile-meta.xml` | `force-app/main/default/profiles/` |
| Remote Site Setting | `RemoteSiteSetting` | `.remoteSite-meta.xml` | `force-app/main/default/remoteSiteSettings/` |
| CSP Trusted Site | `CspTrustedSite` | `.cspTrustedSite-meta.xml` | `force-app/main/default/cspTrustedSites/` |

---

## Integration

| Type | Metadata API Name | File Extension | Folder |
|---|---|---|---|
| Named Credential | `NamedCredential` | `.namedCredential-meta.xml` | `force-app/main/default/namedCredentials/` |
| External Credential | `ExternalCredential` | `.externalCredential-meta.xml` | `force-app/main/default/externalCredentials/` |

---

## Flows and Automation

| Type | Metadata API Name | File Extension | Folder |
|---|---|---|---|
| Flow | `Flow` | `.flow-meta.xml` | `force-app/main/default/flows/` |
| Custom Metadata Type (definition) | `CustomObject` | `.object-meta.xml` | `force-app/main/default/objects/<TypeName__mdt>/` |
| Custom Metadata Record | `CustomMetadata` | `.md-meta.xml` | `force-app/main/default/customMetadata/` |
| Custom Setting (definition) | `CustomObject` | `.object-meta.xml` | `force-app/main/default/objects/<SettingName__c>/` |

---

## UI

| Type | Metadata API Name | File Extension | Folder |
|---|---|---|---|
| FlexiPage | `FlexiPage` | `.flexipage-meta.xml` | `force-app/main/default/flexipages/` |
| Custom Tab | `CustomTab` | `.tab-meta.xml` | `force-app/main/default/tabs/` |
| Custom Application | `CustomApplication` | `.app-meta.xml` | `force-app/main/default/applications/` |
| Page Layout | `Layout` | `.layout-meta.xml` | `force-app/main/default/layouts/` |
| Compact Layout | `CompactLayout` | `.compactLayout-meta.xml` | `force-app/main/default/objects/<ObjectName>/compactLayouts/` |
| Static Resource | `StaticResource` | `.resource` + `.resource-meta.xml` | `force-app/main/default/staticresources/` |

---

## package.xml Snippet — Common Types

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>*</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>*</members>
        <name>ApexTrigger</name>
    </types>
    <types>
        <members>*</members>
        <name>CustomObject</name>
    </types>
    <types>
        <members>*</members>
        <name>LightningComponentBundle</name>
    </types>
    <types>
        <members>*</members>
        <name>Flow</name>
    </types>
    <types>
        <members>*</members>
        <name>PermissionSet</name>
    </types>
    <version>62.0</version>
</Package>
```
