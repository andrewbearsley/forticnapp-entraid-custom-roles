# FortiCNAPP + Entra ID: scoped access with custom roles

How to set up Entra ID SAML SSO with FortiCNAPP so users only see the cloud resources they're responsible for.

## The problem

You've got 100+ AWS accounts and Azure subscriptions in FortiCNAPP. Different people own different parts of the estate. When a technical owner logs in via SSO, they should see their stuff, not the whole lot.

You're not going to maintain a list of account IDs per person. Tag-based resource groups are the way to go here. Filter on a `technical-owner` tag (or whatever your org uses) and let FortiCNAPP pick up the right resources automatically.

## How it works

```
Entra ID Security Group
    ↓ SAML assertion
    ↓ laceworkCustomUserGroups = "<GUID>"
    ↓
FortiCNAPP Custom User Group (GUID)
    ├── Role: Read-Only User (or Admin, Power User)
    └── Resource Group: tag "technical-owner" = "alice@example.com"
            ├── (automatically includes all matching AWS accounts)
            └── (automatically includes all matching Azure subscriptions)
```

The SAML attribute that makes this work is `laceworkCustomUserGroups`. It assigns the SSO user to a pre-configured user group in FortiCNAPP that has both a role and resource group scoping attached.

> **Why not `laceworkAdminRoleAccounts` / `laceworkUserRoleAccounts`?** Those built-in attributes only scope by FortiCNAPP sub-account name. They can't scope by resource group, so you can't use tags or other filters. For anything beyond "this user gets admin on everything", you need custom user groups.

## Prerequisites

- FortiCNAPP org with multiple cloud accounts/subscriptions already integrated
- Entra ID tenant with admin access
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/611253/microsoft-entra-id-saml-sso" target="_blank">SAML SSO between Entra ID and FortiCNAPP</a> already configured (or set it up as part of this)
- <a href="https://docs.lacework.net/cli" target="_blank">Lacework CLI</a> installed (for CLI examples)

## Step 1: Create resource groups in FortiCNAPP

Resource groups control which cloud resources a user can see. You define filters (tags, account ID, subscription, region) and FortiCNAPP matches resources against them.

### Use tags, not account ID lists

With 100+ accounts, you don't want to maintain explicit account lists per team. Tag your AWS accounts and Azure subscriptions with something like `technical-owner`, then create resource groups that filter on that tag. When accounts get tagged or re-assigned, visibility updates on its own.

### Option A: CLI (interactive)

```bash
lacework resource-group create
```

This is interactive. It'll prompt you for the type, name, and description, then drop you into your editor with a JSON query template to fill in.

**AWS, filter by `technical-owner` tag:**

```json
{
  "filters": {
    "owner": {
      "field": "Resource Tag",
      "operation": "EQUALS",
      "key": "technical-owner",
      "values": ["alice@example.com"]
    }
  },
  "expression": {
    "operator": "AND",
    "children": [
      { "filterName": "owner" }
    ]
  }
}
```

**Azure, same idea:**

```json
{
  "filters": {
    "owner": {
      "field": "Resource Tag",
      "operation": "EQUALS",
      "key": "technical-owner",
      "values": ["alice@example.com"]
    }
  },
  "expression": {
    "operator": "AND",
    "children": [
      { "filterName": "owner" }
    ]
  }
}
```

You can combine filters. Tag + region, for example, if you only want to show an owner's resources in certain regions:

```json
{
  "filters": {
    "owner": {
      "field": "Resource Tag",
      "operation": "EQUALS",
      "key": "technical-owner",
      "values": ["alice@example.com"]
    },
    "regions": {
      "field": "Region",
      "operation": "EQUALS",
      "values": ["ap-southeast-2", "us-east-1"]
    }
  },
  "expression": {
    "operator": "AND",
    "children": [
      { "filterName": "owner" },
      { "filterName": "regions" }
    ]
  }
}
```

<details>
<summary><strong>Alternative: explicit account IDs (fine for a handful of accounts)</strong></summary>

If you only need to scope to a few specific accounts, you can list them directly. Gets tedious fast though.

**AWS, specific account IDs:**

```json
{
  "filters": {
    "accounts": {
      "field": "Account",
      "operation": "EQUALS",
      "values": ["111111111111", "222222222222"]
    }
  },
  "expression": {
    "operator": "AND",
    "children": [
      { "filterName": "accounts" }
    ]
  }
}
```

**Azure, specific subscription IDs:**

```json
{
  "filters": {
    "subscriptions": {
      "field": "Subscription ID",
      "operation": "EQUALS",
      "values": ["0fe75302-1906-45ec-bde1-79b76899dd74"]
    }
  },
  "expression": {
    "operator": "AND",
    "children": [
      { "filterName": "subscriptions" }
    ]
  }
}
```

</details>

Grab the resource group ID after creation. You'll need it later:

```bash
lacework resource-group list
```

### Option B: Terraform

```hcl
# Tag-based: filter by technical-owner tag
resource "lacework_resource_group" "alice_aws" {
  name        = "Alice - AWS Resources"
  type        = "AWS"
  description = "AWS resources owned by alice@example.com"

  group {
    operator = "AND"
    filter {
      filter_name = "owner"
      field       = "Resource Tag"
      key         = "technical-owner"
      operation   = "EQUALS"
      value       = ["alice@example.com"]
    }
  }
}

resource "lacework_resource_group" "alice_azure" {
  name        = "Alice - Azure Resources"
  type        = "AZURE"
  description = "Azure resources owned by alice@example.com"

  group {
    operator = "AND"
    filter {
      filter_name = "owner"
      field       = "Resource Tag"
      key         = "technical-owner"
      operation   = "EQUALS"
      value       = ["alice@example.com"]
    }
  }
}
```

<details>
<summary><strong>Alternative: explicit account/subscription IDs</strong></summary>

```hcl
resource "lacework_resource_group" "team_a_aws" {
  name        = "Team A AWS Accounts"
  type        = "AWS"
  description = "AWS accounts managed by Team A"

  group {
    operator = "AND"
    filter {
      filter_name = "accounts"
      field       = "Account"
      operation   = "EQUALS"
      value       = ["111111111111", "222222222222"]
    }
  }
}

resource "lacework_resource_group" "team_a_azure" {
  name        = "Team A Azure Subscriptions"
  type        = "AZURE"
  description = "Azure subscriptions managed by Team A"

  group {
    operator = "AND"
    filter {
      filter_name = "subscriptions"
      field       = "Subscription ID"
      operation   = "EQUALS"
      value       = ["0fe75302-1906-45ec-bde1-79b76899dd74"]
    }
  }
}
```

</details>

### Option C: Console UI

Settings > Resource Groups > + Create. Pick the resource type, build your filter conditions in the visual editor, save.

### Filter field reference

| Resource Type | Fields |
|--------------|--------|
| AWS | `Account`, `Region`, `Resource Tag` |
| Azure | `Subscription ID`, `Subscription Name`, `Tenant ID`, `Tenant Name`, `Region`, `Resource Tag` |
| GCP | `Organization`, `Folder`, `Project`, `Region`, `Resource Tag` |

Operations: `EQUALS`, `STARTS_WITH`, `INCLUDES`. You can nest `AND`/`OR` expressions up to 3 levels deep.

## Step 2: Create custom user groups in FortiCNAPP

A custom user group ties together a role (what the user can do) and one or more resource groups (what they can see).

1. Settings > Access Control > User Groups
2. Add User Group
3. Give it a name (e.g. `Alice Read-Only`)
4. Pick a role: `Read-only User`, `Power User`, `Admin`, or a custom role
5. Assign the resource groups you created in step 1
6. Save

Copy the user group GUID. You need this for the SAML claim. It's in the user group details, or you can pull it via:

```bash
lacework api get /api/v2/TeamUsers --json
```

Create one of these for each access pattern you need (e.g. `Alice Read-Only`, `Platform Team Admin`, `Azure Team Power User`).

## Step 3: Configure the Entra ID enterprise application

If you haven't set up SAML SSO yet, follow the <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/611253/microsoft-entra-id-saml-sso" target="_blank">Entra ID SAML SSO guide</a>. Short version:

1. Entra ID > Enterprise Applications > New Application (non-gallery) for FortiCNAPP
2. Single sign-on > SAML. Set the Entity ID and Reply URL (ACS) from FortiCNAPP (Settings > Authentication > SAML)
3. Download the Federation Metadata XML

## Step 4: Configure SAML claims in Entra ID

This is where Entra ID groups get wired to FortiCNAPP custom user groups.

### Add the claim

In the enterprise application:

1. Single sign-on > Attributes & Claims > Edit
2. Add new claim
3. Name: `laceworkCustomUserGroups`
4. Source: Attribute
5. Source attribute: the FortiCNAPP user group GUID(s)

### How to map the values

Two approaches depending on how many teams you're dealing with:

**User extension attributes** work well for a small-medium number of teams. Store the FortiCNAPP GUID(s) as a custom attribute on the user in Entra ID (e.g. `extension_laceworkCustomUserGroups`), populate it with comma-separated GUIDs, and map the claim to `user.extension_laceworkCustomUserGroups`.

**Group-based claims with transforms** scale better. Create Entra ID security groups (e.g. `FortiCNAPP-TeamA-ReadOnly`) and use claim conditions to emit different GUID values based on which group the user belongs to.

### Multiple user groups

If someone needs access across multiple scopes, comma-separate the GUIDs:

```
<GUID-1>,<GUID-2>
```

## Step 5: Enable JIT provisioning in FortiCNAPP

JIT (Just-In-Time) provisioning creates user accounts automatically on first SSO login, so you don't have to pre-create them in FortiCNAPP.

1. Settings > Authentication > SAML
2. Add New (or edit existing)
3. Upload the Entra ID Federation Metadata XML
4. Name the identity provider
5. Turn on Just-In-Time User Provisioning
6. Save

Note: if OAuth is currently enabled, you need to disable it first.

## Step 6: Test it

1. Assign a test user to the relevant Entra ID security group
2. Have them log in via SSO
3. Check that:
   - Their account was created automatically (JIT)
   - They're in the correct custom user group
   - They can only see resources in their assigned resource groups
   - They can't see anything outside their scope

Verify from the CLI:

```bash
lacework api get /api/v2/TeamUsers --json | jq '.data[] | select(.userName == "test@example.com")'
```

## Examples

### Scenario A: Technical owner, read-only on their tagged resources

| Component | Value |
|-----------|-------|
| Resource Group | `Alice - AWS Resources` (Tag: `technical-owner` = `alice@example.com`) |
| Custom User Group | `Alice Read-Only`, Role: Read-only, Resource Group: above |
| Entra ID Group | `FortiCNAPP-Alice-ReadOnly` |
| SAML Claim | `laceworkCustomUserGroups` = `<alice-readonly-guid>` |

When new accounts get tagged `technical-owner: alice@example.com`, they show up in Alice's view automatically.

### Scenario B: Technical owner spanning AWS and Azure

| Component | Value |
|-----------|-------|
| Resource Groups | `Bob - AWS Resources` (tag filter) + `Bob - Azure Resources` (tag filter) |
| Custom User Group | `Bob Read-Only`, Role: Read-only, Resource Groups: both above |
| Entra ID Group | `FortiCNAPP-Bob-ReadOnly` |
| SAML Claim | `laceworkCustomUserGroups` = `<bob-readonly-guid>` |

One custom user group can reference multiple resource groups across cloud providers.

### Scenario C: Platform team, admin on everything

| Component | Value |
|-----------|-------|
| Custom User Group | Not needed, use built-in role attribute |
| SAML Claim | `laceworkAdminRoleAccounts` = `*` |

For full admin on everything, the built-in SAML attribute is simpler. No need for custom user groups or resource groups.

### Scenario D: Team-based, power user on a team's subscriptions

| Component | Value |
|-----------|-------|
| Resource Group | `Platform Team - Azure` (Tag: `team` = `platform`) |
| Custom User Group | `Platform Azure Power Users`, Role: Power User, Resource Group: above |
| Entra ID Group | `FortiCNAPP-Platform-AzurePU` |
| SAML Claim | `laceworkCustomUserGroups` = `<platform-azure-pu-guid>` |

## SAML JIT attribute reference

| Attribute | Value | Purpose |
|-----------|-------|---------|
| `laceworkCustomUserGroups` | Comma-separated GUIDs | Assign to custom user groups (with resource group scoping) |
| `laceworkAdminRoleAccounts` | Account names or `*` | Built-in Admin role on specified sub-accounts |
| `laceworkPowerUserRoleAccounts` | Account names or `*` | Built-in Power User role on specified sub-accounts |
| `laceworkUserRoleAccounts` | Account names or `*` | Built-in Read-only role on specified sub-accounts |
| `laceworkOrgAdminRole` | `true` / `false` | Organization Administrator (all accounts, all permissions) |
| `laceworkOrgUserRole` | `true` / `false` | Organization User (all accounts, read-only org settings) |

Precedence if a user ends up with multiple roles on the same account: Org Admin > Admin > Power User > Read-only. Highest wins.

## Links

- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/393390/microsoft-entra-id-saml-jit" target="_blank">Entra ID SAML JIT</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/611253/microsoft-entra-id-saml-sso" target="_blank">Entra ID SAML SSO</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/698623/resource-groups" target="_blank">Resource groups</a>
- <a href="https://docs.fortinet.com/document/lacework-forticnapp/25.2.0/administration-guide/873733/access-control-overview" target="_blank">Access control overview</a>
- <a href="https://registry.terraform.io/providers/lacework/lacework/latest/docs/resources/resource_group" target="_blank">lacework_resource_group (Terraform)</a>
- <a href="https://docs.lacework.net/cli/commands/lacework_resource-group" target="_blank">lacework resource-group (CLI)</a>
