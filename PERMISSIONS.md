# Permission Model Design

Designing Permission Sets, Custom Permissions, and feature access control for Salesforce orgs.

---

## Permission Sets vs. Profiles

### Profiles (Legacy)

A profile is a mandatory, hard-to-change role for every user.

- Each user has exactly one profile
- Includes login hours, IP restrictions, record types, field permissions
- Changes to a profile affect all users with that profile
- Hard to test (can't assign multiple profiles to test edge cases)

**Avoid for new features.** Use Permission Sets instead.

### Permission Sets (Modern)

Permission Sets are layered on top of a profile. A user can have multiple Permission Sets.

- User has 1 Profile + 0 to many Permission Sets
- Easy to test (assign/remove on demand)
- Granular control (can give one user a feature without changing their profile)
- Changes affect only assigned users

**Use Permission Sets for all new features.**

---

## Building a Permission Set for a Feature

### Step 1: Identify What the User Needs

A feature might require:
- Access to custom objects (Create, Read, Update, Delete)
- Field-level access (read, edit)
- Custom permissions
- Tab visibility
- Apex class access
- Page/Component access
- Custom settings access

### Step 2: Create the Permission Set

Setup > Users > Permission Sets → New

```
Name: Opportunity_Manager
Description: Allows management of opportunities and related data
License: No License (admin use) OR Standard User (for feature)
```

### Step 3: Add Permissions

#### Object Permissions

Setup > Permission Sets > [PermissionSet] > Object Settings

| Object | Create | Read | Update | Delete |
|--------|--------|------|--------|--------|
| Account | ✓ | ✓ | ✓ | ✗ |
| Opportunity | ✓ | ✓ | ✓ | ✓ |
| OpportunityLineItem | ✗ | ✓ | ✗ | ✗ |

#### Field Permissions

Setup > Permission Sets > [PermissionSet] > Field Permissions

For each custom field, grant Read and/or Edit access.

**Important Rule**: If a field is required (required=true in metadata), do NOT include it in fieldPermissions. Salesforce throws an error if you try to assign fieldPermissions for required fields.

```xml
<!-- CORRECT: Required fields not in fieldPermissions -->
<fieldPermissions>
  <editable>true</editable>
  <field>Opportunity.CustomField__c</field>
  <readable>true</readable>
</fieldPermissions>
<!-- Required field, skip it -->

<!-- WRONG: Including required field -->
<fieldPermissions>
  <editable>true</editable>
  <field>Opportunity.StageName</field>  <!-- Required, will error -->
  <readable>true</readable>
</fieldPermissions>
```

#### Tab Settings

Setup > Permission Sets > [PermissionSet] > Tab Settings

| Tab | Visibility |
|-----|------------|
| Opportunities | Available |
| Custom Tab | Available |

Enum values: Available, Hidden, None

#### Custom Permissions

Setup > Permission Sets > [PermissionSet] > Custom Permissions

| Permission | Enabled |
|------------|---------|
| Can_Approve_Deals | ✓ |
| Can_Export_Data | ✗ |

#### Apex Class Access

Setup > Permission Sets > [PermissionSet] > Apex Class Access

| Class | Enabled |
|-------|---------|
| OpportunityApprovalService | ✓ |
| ReportService | ✓ |

---

## Permission Set XML Structure

A Permission Set is deployed as XML.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
  <description>Allows management of opportunities</description>
  <label>Opportunity_Manager</label>
  
  <!-- Object permissions -->
  <objectPermissions>
    <allowCreate>true</allowCreate>
    <allowDelete>true</allowDelete>
    <allowEdit>true</allowEdit>
    <allowRead>true</allowRead>
    <modifyAllRecords>false</modifyAllRecords>
    <object>Opportunity</object>
    <viewAllRecords>false</viewAllRecords>
  </objectPermissions>
  
  <!-- Field permissions (non-required fields only) -->
  <fieldPermissions>
    <editable>true</editable>
    <field>Opportunity.CustomField__c</field>
    <readable>true</readable>
  </fieldPermissions>
  
  <!-- Tab visibility -->
  <tabSettings>
    <tab>Opportunity_Custom_App</tab>
    <visibility>Available</visibility>
  </tabSettings>
  
  <!-- Apex class access -->
  <classAccesses>
    <apexClass>OpportunityApprovalService</apexClass>
    <enabled>true</enabled>
  </classAccesses>
  
  <!-- Custom permission -->
  <customPermissions>
    <enabled>true</enabled>
    <name>Can_Approve_Deals</name>
  </customPermissions>
</PermissionSet>
```

---

## Testing Permission Models

### Pattern: Test with Limited User

```apex
@IsTest
private class OpportunityPermissionTest {
  @TestSetup
  static void setupUsers() {
    // Create standard user
    User limitedUser = new User(
      FirstName = 'Limited',
      LastName = 'User',
      Email = 'limited@example.com',
      Username = 'limited@example.com.' + System.now().millisecond(),
      ProfileId = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1].Id,
      Alias = 'lim1',
      TimeZoneSidKey = 'America/Los_Angeles',
      LocaleSidKey = 'en_US',
      EmailEncodingKey = 'UTF-8',
      LanguageLocaleKey = 'en_US'
    );
    insert limitedUser;
  }
  
  @IsTest
  static void testOpportunityManagerPermissions() {
    User limitedUser = [SELECT Id FROM User WHERE Email = 'limited@example.com' LIMIT 1];
    
    // Assign permission set
    PermissionSetAssignment psa = new PermissionSetAssignment(
      PermissionSetId = [SELECT Id FROM PermissionSet WHERE Name = 'Opportunity_Manager'].Id,
      AssigneeId = limitedUser.Id
    );
    insert psa;
    
    // Test as limited user
    System.runAs(limitedUser) {
      // Should be able to create opportunity
      Opportunity opp = new Opportunity(Name = 'Test Deal', StageName = 'Prospecting', CloseDate = Date.today());
      insert opp;
      
      // Should be able to read
      Opportunity retrieved = [SELECT Id FROM Opportunity WHERE Id = :opp.Id];
      System.assert(retrieved != null);
      
      // Should be able to update
      retrieved.Name = 'Updated Deal';
      update retrieved;
      
      // Should be able to delete
      delete retrieved;
    }
  }
  
  @IsTest
  static void testWithoutPermission() {
    User limitedUser = [SELECT Id FROM User WHERE Email = 'limited@example.com' LIMIT 1];
    
    // Don't assign permission set
    
    System.runAs(limitedUser) {
      // Should not be able to create opportunity
      Opportunity opp = new Opportunity(Name = 'Test Deal', StageName = 'Prospecting', CloseDate = Date.today());
      
      try {
        insert opp;
        System.assert(false, 'Should not have create permission');
      } catch (DmlException ex) {
        System.assert(ex.getMessage().contains('INSUFFICIENT_ACCESS'));
      }
    }
  }
}
```

---

## Custom Permissions

Custom permissions are Boolean flags you define. Use them to gate access to features.

### Setup Custom Permission

Setup > Custom Code > Custom Permissions → New

```
Label: Can Approve Deals
Name: Can_Approve_Deals
Description: Controls access to deal approval feature
```

### Assign to Permission Set

Setup > Permission Sets > [PermissionSet] > Custom Permissions

Check "Can_Approve_Deals".

### Check in Apex

```apex
public class DealApprovalService {
  public static void approveDeal(Id dealId) {
    // Check permission
    if (!FeatureManagement.checkPermission('Can_Approve_Deals')) {
      throw new SecurityException('You do not have permission to approve deals');
    }
    
    // Process approval
    Deal__c deal = [SELECT Id FROM Deal__c WHERE Id = :dealId];
    deal.Status__c = 'Approved';
    update deal;
  }
}
```

---

## Role Hierarchy

Role hierarchy controls data visibility in reports and list views (not CRUD).

Setup > Users > Roles

Create hierarchy:

```
CEO
├── VP Sales
│  ├── Sales Manager 1
│  │  ├── Sales Rep 1
│  │  └── Sales Rep 2
│  └── Sales Manager 2
└── VP Operations
   ├── Finance Manager
   └── Analyst
```

**Impact**: Users can see records owned by their subordinates in reports.

**Does NOT control**: Who can create/edit records. Use OWD and Sharing Rules for that.

---

## Org-Wide Defaults (OWD)

OWD sets the baseline sharing level for records.

Setup > Feature Settings > Sharing Settings

| Object | Setting |
|--------|---------|
| Account | Private |
| Opportunity | Controlled by Parent (Account) |
| Task | Public Read/Write |

### Sharing Rules

If OWD is Private, use Sharing Rules to grant access.

Setup > Feature Settings > Sharing Settings → [Object] Sharing Rules

```
Rule Name: Sales Managers View All Accounts
User: Sales Manager role
Account Access Level: Read Only
```

When a sales manager views a report, they see:
- Accounts they own
- Accounts owned by their subordinates (role hierarchy)
- Accounts explicitly shared with them (sharing rules)

---

## Permission Set Group

Permission Set Groups bundle multiple Permission Sets.

Setup > Users > Permission Set Groups → New

```
Name: Opportunity_Manager_Suite
Permission Sets to include:
  - Opportunity_Manager
  - Reports_Access
  - Dashboard_Access
```

Assign group to users. All bundled Permission Sets grant access.

---

## Common Mistakes

### Mistake 1: Not Checking Permissions in Code

```apex
// ❌ Wrong (no permission check)
public static void approveDeal(Id dealId) {
  Deal__c deal = [SELECT Id FROM Deal__c WHERE Id = :dealId];
  deal.Status__c = 'Approved';
  update deal;
}

// ✅ Right (checks permission)
public static void approveDeal(Id dealId) {
  if (!FeatureManagement.checkPermission('Can_Approve_Deals')) {
    throw new SecurityException('No permission');
  }
  
  Deal__c deal = [SELECT Id FROM Deal__c WHERE Id = :dealId];
  deal.Status__c = 'Approved';
  update deal;
}
```

### Mistake 2: Including Required Fields in fieldPermissions

```xml
<!-- ❌ Wrong (StageName is required) -->
<fieldPermissions>
  <field>Opportunity.StageName</field>
  <editable>true</editable>
  <readable>true</readable>
</fieldPermissions>

<!-- ✅ Right (omit required fields) -->
<fieldPermissions>
  <field>Opportunity.CustomField__c</field>
  <editable>true</editable>
  <readable>true</readable>
</fieldPermissions>
```

### Mistake 3: Not Testing with Limited User

```apex
// ❌ Wrong (tests as admin, no permission issues caught)
@IsTest
static void testDealApproval() {
  Opportunity opp = new Opportunity(...);
  insert opp;
  DealApprovalService.approveDeal(opp.Id);  // Runs as admin
}

// ✅ Right (tests with limited user)
@IsTest
static void testDealApproval() {
  User limitedUser = [SELECT Id FROM User WHERE Email = 'limited@example.com'];
  
  System.runAs(limitedUser) {
    try {
      DealApprovalService.approveDeal(opportunityId);
      System.assert(false, 'Should fail without permission');
    } catch (SecurityException ex) {
      // Expected
    }
  }
}
```

### Mistake 4: Over-Permissioning

```apex
// ❌ Wrong (giving admin permission to everyone)
Profile: Standard User
PermissionSet: Can_Do_Everything  // Too broad

// ✅ Right (minimal permissions)
Profile: Standard User
PermissionSet: Opportunity_Editor  // Only what's needed
```

---

## Checklist: Permission Model Ready for Production

- ✅ All custom objects have Permission Sets (not Profile-based)
- ✅ Custom fields have fieldPermissions (except required fields)
- ✅ Object permissions set (Create, Read, Update, Delete)
- ✅ Custom permissions for sensitive operations
- ✅ Permission checks in code (FeatureManagement.checkPermission)
- ✅ Tested with limited users (System.runAs)
- ✅ Role hierarchy in place (if needed)
- ✅ Org-Wide Defaults configured
- ✅ Sharing Rules in place (if OWD is Private)
- ✅ No hardcoded permission assumptions in code
- ✅ Documentation of who needs what access

