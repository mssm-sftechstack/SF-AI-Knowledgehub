# Permission Sets Over Profiles

**Use Permission Sets for everything new. Profiles are legacy. Mixing them causes hard-to-debug access issues.**

## Why Profiles Are Being Phased Out

Profiles are monolithic. One profile controls an entire user's access to objects, fields, apps, page layouts, and login settings. Changing one thing means editing a large metadata file that's shared across everyone on that profile.

Salesforce's roadmap is moving toward a Permission Set-only model. New features (like Permission Set Groups) are PS-only. Profiles won't disappear overnight, but no new feature logic should depend on them.

The practical problem: when you grant access in a Profile and restrict it in a PS (or vice versa), the most permissive wins for some things and the most restrictive for others. It's non-obvious and creates access bugs that are tedious to trace.

## Permission Set Structure

One PS per feature. A user gets the features they need by getting the right Permission Sets assigned.

```
Feature: Case Management
  → PS: Case_Management_User
      - Object: Case (Read, Create, Edit)
      - Field: Case.InternalNotes__c (Read, Edit)
      - Custom Permission: Can_Escalate_Case
      - Tab: Cases (Available)

Feature: Admin Tools
  → PS: Admin_Tools
      - Object: All objects (Read, Create, Edit, Delete)
      - Custom Permission: Bypass_AccountTrigger
```

A user who needs both features gets both PSes assigned. No profile editing required.

## What Goes in a PS

- Object CRUD permissions
- Field-level security (read/edit per field)
- Custom Permissions
- Tab visibility
- App permissions
- Apex class access
- VF page access

## What NEVER Goes in a PS

**Required field `fieldPermissions`.**

If a field is defined with `required=true` in its `.field-meta.xml`, don't include it in `fieldPermissions` in any Permission Set. Salesforce throws a deploy error:

```
Error: You cannot deploy to a required field. Remove the fieldPermissions for
CustomField__c from your PermissionSet.
```

Required fields are always readable and editable by users with object access. You don't need to grant access, and you can't restrict it via FLS. Exclude them entirely.

### How to check

```xml
<!-- force-app/main/default/objects/Case__c/fields/Status__c.field-meta.xml -->
<CustomField>
    <fullName>Status__c</fullName>
    <required>true</required>   <!-- if this is true, exclude from PS fieldPermissions -->
    ...
</CustomField>
```

Read the field metadata before writing the PS. Don't assume.

## Custom Permissions

Define a Custom Permission to gate features that aren't controlled by object/field access, like bypassing a trigger, enabling a beta feature, or unlocking admin-only UI.

```xml
<!-- force-app/main/default/customPermissions/Can_Escalate_Case.customPermission-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<CustomPermission xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>Allows the user to escalate a case to Tier 2 support</description>
    <label>Can Escalate Case</label>
</CustomPermission>
```

Check it in Apex with:

```apex
if (FeatureManagement.checkPermission('Can_Escalate_Case')) {
    // show escalate button or allow escalation logic
}
```

Use Custom Permissions for bypass patterns (like skipping trigger logic for data loads) instead of static booleans. They're configurable per-user without a code deploy.

## Permission Set Metadata Structure

```xml
<!-- force-app/main/default/permissionsets/Case_Management_User.permissionset-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Case Management User</label>
    <hasActivationRequired>false</hasActivationRequired>

    <!-- Object permissions -->
    <objectPermissions>
        <allowCreate>true</allowCreate>
        <allowDelete>false</allowDelete>
        <allowEdit>true</allowEdit>
        <allowRead>true</allowRead>
        <modifyAllRecords>false</modifyAllRecords>
        <object>Case</object>
        <viewAllRecords>false</viewAllRecords>
    </objectPermissions>

    <!-- Field permissions — only for non-required fields -->
    <fieldPermissions>
        <editable>true</editable>
        <field>Case.InternalNotes__c</field>
        <readable>true</readable>
    </fieldPermissions>

    <!-- Custom permissions -->
    <customPermissions>
        <enabled>true</enabled>
        <name>Can_Escalate_Case</name>
    </customPermissions>

    <!-- Tab settings -->
    <tabSettings>
        <tab>standard-Case</tab>
        <visibility>Available</visibility>
    </tabSettings>
</PermissionSet>
```

Key points about the metadata:
- `tabSettings` not `tabVisibilities` — the correct element name for PS metadata
- Tab visibility values are `Available` or `None` (not `DefaultOn`)
- Elements must appear in the order the XSD expects: `objectPermissions` before `fieldPermissions` before `customPermissions` before `tabSettings`
