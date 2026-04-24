# Foundry Governance: Block Basic Agent Setup

Azure Policy and Custom RBAC controls to prevent users from creating agents using **basic agent setup** (Microsoft-managed resources) in Microsoft Foundry projects.

## Problem Statement

When Foundry projects are created (e.g., via PAM accounts), they are provisioned **without capability hosts**. This allows users to create agents using **basic agent setup**, which uses Microsoft-managed multitenant resources (Cosmos DB, Storage, AI Search) — outside the organization's control.

Organizations that require **standard agent setup** (customer-owned resources) need a way to:

1. **Prevent** new Foundry projects from being created without customer-owned storage (basic setup)
2. **Block** agent API operations on existing projects that lack standard setup

## Basic vs Standard Agent Setup

| Feature | Basic Setup | Standard Setup |
|---|---|---|
| **Infrastructure** | Microsoft-managed multitenant | Customer-owned dedicated |
| **Capability Hosts** | Not required | Required |
| **Storage** | Microsoft-managed | Customer-owned (via `userOwnedStorage`) |
| **Cosmos DB** | Microsoft-managed | Customer-owned |
| **AI Search** | Microsoft-managed | Customer-owned |
| **Data Residency** | Microsoft-controlled | Customer-controlled |
| **Network Isolation** | Not available | Available (Private Endpoints) |

## Solution Overview

There is **no built-in Azure toggle** to selectively disable basic agent setup. This repo provides two complementary controls:

| Control | Scope | What It Blocks |
|---|---|---|
| **Azure Policy** | Control plane (ARM) | Creating new Foundry projects without customer-owned storage |
| **Custom RBAC Role** | Data plane (API) | Agent API calls on existing projects |

> **Recommendation**: Use both controls together for defense in depth.

---

## 1. Azure Policy: Block Basic Setup Projects

### How It Works

This policy **denies** the creation or update of `Microsoft.CognitiveServices/accounts` (kind: `AIServices`) that have zero `userOwnedStorage` entries — effectively blocking Foundry projects configured for basic setup.

### Policy Definition

File: [`policies/block-basic-agents-policy.json`](policies/block-basic-agents-policy.json)

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.CognitiveServices/accounts"
        },
        {
          "field": "kind",
          "equals": "AIServices"
        },
        {
          "count": {
            "field": "Microsoft.CognitiveServices/accounts/userOwnedStorage[*]"
          },
          "equals": 0
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

### Deploy via Azure Portal

1. Go to **Azure Policy** → **Definitions** → **+ Policy definition**
2. Set **Definition location** to your subscription or management group
3. Set **Name**: `Block Foundry Projects Without Standard Setup`
4. Set **Category**: Create new → `AI Foundry`
5. Paste the policy rule JSON into the editor
6. Click **Save**
7. Go to **Assignments** → assign the policy to your target scope

### Deploy via Azure CLI

```bash
# Create the policy definition
az policy definition create \
  --name "block-foundry-basic-setup" \
  --display-name "Block Foundry Projects Without Standard Setup" \
  --description "Denies creation of AI Services accounts (Foundry projects) without customer-owned storage, preventing basic agent setup." \
  --rules policies/block-basic-agents-policy.json \
  --mode All

# Assign the policy to a subscription
az policy assignment create \
  --name "block-foundry-basic-setup" \
  --policy "block-foundry-basic-setup" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```

### Limitations

- Azure Policy operates on the **control plane** only — it blocks resource creation/updates via ARM
- It does **not** block agent API calls (data plane) on projects that already exist
- Use the Custom RBAC Role below to cover existing projects

---

## 2. Custom RBAC Role: Block Agent Operations

### How It Works

This custom role grants access to all Foundry AI Services data operations **except** agent operations. Users assigned this role can use Foundry features (models, deployments, etc.) but cannot create or manage agents.

To selectively enable agents on projects with standard setup, assign the built-in **Azure AI User** role at the project scope.

### Role Definition

File: [`roles/foundry-no-agents-role.json`](roles/foundry-no-agents-role.json)

```json
{
  "Name": "Foundry User - No Agents",
  "Description": "Grants Foundry access but blocks all Agent Service operations. Assign Azure AI User at project scope after standard setup to selectively enable agents.",
  "IsCustom": true,
  "AssignableScopes": [
    "/subscriptions/<SUBSCRIPTION_ID>"
  ],
  "Actions": [
    "Microsoft.CognitiveServices/*/read",
    "Microsoft.Authorization/*/read",
    "Microsoft.CognitiveServices/accounts/listkeys/action",
    "Microsoft.Resources/deployments/*"
  ],
  "NotActions": [],
  "DataActions": [
    "Microsoft.CognitiveServices/accounts/AIServices/*"
  ],
  "NotDataActions": [
    "Microsoft.CognitiveServices/accounts/AIServices/agents/*"
  ]
}
```

### Deploy via Azure CLI

```bash
# Update AssignableScopes in the JSON file first, then:
az role definition create --role-definition roles/foundry-no-agents-role.json
```

### Workflow: Selectively Enable Agents

```
1. Assign "Foundry User - No Agents" at subscription/RG scope (blocks agents everywhere)
2. When a project is confirmed to have standard setup (capability hosts + customer-owned resources):
   az role assignment create \
     --assignee <USER_OR_GROUP> \
     --role "Azure AI User" \
     --scope "/subscriptions/<SUB>/resourceGroups/<RG>/providers/Microsoft.CognitiveServices/accounts/<PROJECT>"
3. The Azure AI User role at project scope overrides the NotDataActions, enabling agents for that project
```

### Prerequisites

- **Owner** or **User Access Administrator** role on the subscription (required to create custom role definitions)
- Custom role definitions are written at the subscription level regardless of `AssignableScopes`

---

## Repository Structure

```
├── README.md
├── policies/
│   └── block-basic-agents-policy.json    # Azure Policy definition
└── roles/
    └── foundry-no-agents-role.json       # Custom RBAC role definition
```

## Additional Considerations

### Audit Existing Projects

To identify existing Foundry projects using basic setup:

```bash
az resource list \
  --resource-type "Microsoft.CognitiveServices/accounts" \
  --query "[?kind=='AIServices' && (properties.userOwnedStorage==null || length(properties.userOwnedStorage)==\`0\`)].{name:name, rg:resourceGroup}" \
  -o table
```

### PIM/PAM Considerations

- If your organization uses **Privileged Identity Management (PIM)**, Owner roles may be **eligible** but not **active**
- You must **activate** the Owner role in PIM before running role definition commands
- Path: Azure Portal → Entra ID → PIM → My roles → Azure resources → Activate

## License

MIT
