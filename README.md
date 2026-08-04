# MapleTech Secure Foundation — Azure Policy Lab

## Summary

This lab implements a set of governance guardrails for MapleTech using Azure Policy. The objective was to prevent non-compliant resource deployments *before* they happen, rather than detecting and remediating them afterward.

Three custom Azure Policies were created, each with a **Deny** effect:

1. Restrict resource deployment to the **Canada Central** region only
2. Require every resource to carry a **ProjectName** tag
3. Block the creation of **public IP addresses**

These three policies were grouped into a single **Policy Initiative** named **MapleTech Secure Foundation**, categorized under Compliance. The initiative was then assigned to a target resource group with policy enforcement set to **Default (Enforce)**, meaning violations are actively blocked at deployment time rather than just logged.

Enforcement was validated with four test deployments — three intentionally non-compliant (denied) and one fully compliant (allowed) — confirming that each policy triggers correctly and that the initiative as a whole behaves as expected.

## Explanation of Each Policy

### 1. Only-CanadaCentral

**Effect:** Deny
**Purpose:** Enforces data residency by ensuring resources are only deployed in the Canada Central region.

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "not": {
        "field": "location",
        "equals": "canadacentral"
      }
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

**How it works:** The policy checks the `location` field of every resource being deployed. If the location is anything other than `canadacentral`, the deployment is denied. `mode: All` is used because location applies to virtually every resource type, not just tag-supporting ones.

### 2. Require-ProjectName-Tag

**Effect:** Deny
**Purpose:** Ensures every resource can be traced back to a project for cost tracking and ownership accountability.

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "tags['ProjectName']",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

**How it works:** The policy inspects each resource for a tag named `ProjectName`. If that tag does not exist, the resource is denied. `mode: Indexed` is used so the policy only evaluates resource types that actually support tags, avoiding false evaluations on resource types that don't.

### 3. Deny-Public-IP

**Effect:** Deny
**Purpose:** Reduces external attack surface by preventing any public-facing IP resources from being created.

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "field": "type",
      "equals": "Microsoft.Network/publicIPAddresses"
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

**How it works:** The policy matches on the resource `type` field. Any attempt to create a `Microsoft.Network/publicIPAddresses` resource — whether standalone or attached to another resource like a VM's NIC — is denied.

## Initiative: MapleTech Secure Foundation

All three policies above were grouped into one Policy Initiative so they can be assigned and managed as a single unit rather than individually.

- **Name:** MapleTech Secure Foundation
- **Category:** Compliance
- **Included policies:** Only-CanadaCentral, Require-ProjectName-Tag, Deny-Public-IP
- **Assigned scope:** Resource group
- **Enforcement mode:** Default (Enforce) — non-compliant resources are actively blocked, not just flagged

## Test Results

| Test Case | Region | Tag | Public IP | Result |
|---|---|---|---|---|
| VM in East US | East US | — | — | ❌ Denied (Only-CanadaCentral) |
| Storage account, no tag | Canada Central | Missing | — | ❌ Denied (Require-ProjectName-Tag) |
| Standalone public IP | Canada Central | — | — | ❌ Denied (Deny-Public-IP) |
| VM in Canada Central, tagged, no public IP | Canada Central | ProjectName set | None | ✅ Allowed |

**Note:** For the compliant VM to deploy successfully, its NIC had to be explicitly configured with Public IP = None. Otherwise, the VM creation process would also attempt to provision a public IP, which would be blocked by Deny-Public-IP — even though the VM resource itself was fully compliant.

## Takeaways

- Azure Policy enforces governance at deployment time, preventing non-compliant resources from ever being created rather than requiring cleanup after the fact.
- Grouping policies into an initiative makes them easier to manage, assign, and audit as a single governance unit.
- Policies can interact in non-obvious ways (e.g. a compliant VM still being blocked by an unrelated public IP policy through its network interface), which highlights the importance of testing enforcement end-to-end rather than validating each policy in isolation.
