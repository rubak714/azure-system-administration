# 🛠️ Private Endpoint and Policy Governance

## RequestDisallowedByPolicy Error When Creating Private Endpoints

### The Error

When creating a private endpoint in `rg-storage-lab`, you may encounter this error:

```
RequestDisallowedByPolicy: Resource 'privatelink.blob.core.windows.net' was disallowed by policy

Code: RequestDisallowedByPolicy
Target: storage account and private link DNS
Message: Resource was disallowed by policy 'Require a tag on resources'
Policy Parameter: tagName=StorageType
Scope: /subscriptions/.../resourcegroups/rg-storage-lab
```

### Root Cause

The "Require a tag on resources" policy assigned to `rg-storage-lab` applies to **all resources** created in that RG, including networking resources. When you create a private endpoint in `rg-storage-lab`, Azure attempts to create two resources that require the `StorageType` tag:

1. **The private endpoint object itself** - a networking resource
2. **The private link DNS zone** (`privatelink.blob.core.windows.net`) - a DNS resource

Both fail because they don't have the `StorageType` tag, and the policy blocks their creation before you can tag them.

### Why This Matters

This is a **real-world governance pattern** you will encounter in production Azure environments:

- **Storage teams** create storage accounts with storage-specific policies (tagging, encryption, redundancy)
- **Networking teams** create VNets and private endpoints with networking-specific policies
- When storage and networking live in the same RG, their policies conflict
- Separating RGs is the standard solution

### Solution 1: Separate Resource Groups (Recommended)

**Best practice approach:**

1. **Create a new resource group for networking:**
   ```
   Name: rg-network-lab
   Region: same as your other resources
   ```

2. **Create the VNet in the networking RG:**
   - Resource group: `rg-network-lab`
   - Name: `vnet-storage-lab`
   - Address space: `10.0.0.0/16`
   - Subnet: `subnet-storage` with address range `10.0.1.0/24`

3. **Create the private endpoint in the networking RG:**
   - Resource group: `rg-network-lab`
   - Name: `storage-pe`
   - Resource: your storage account (in `rg-storage-lab`)
   - Target sub-resource: `blob`
   - Virtual network: `vnet-storage-lab`
   - Subnet: `subnet-storage`

**Result:**
- ✓ No policy conflicts (storage policies apply to `rg-storage-lab`, networking policies apply to `rg-network-lab`)
- ✓ Clean separation of concerns
- ✓ Reflects real-world architecture
- ✓ Scales to production environments

**Why this is production-safe:**
- Each RG can have its own compliance policies
- Storage teams control storage governance
- Networking teams control network governance
- Private endpoints can cross RG boundaries without conflict

### Solution 2: Tag Private Endpoint Resources (Quick Fix, Not Recommended)

**If you must keep everything in `rg-storage-lab`:**

1. Create the private endpoint in `rg-storage-lab`
2. At creation time, add the required tags:
   - `StorageType: lab` (satisfies the policy)
   - `cost center: your-value` (if that policy exists)

**Drawbacks:**
- ✗ Mixes networking resources with storage resources
- ✗ Not a production pattern
- ✗ Harder to manage compliance across teams
- ✗ Doesn't scale when more policies are added
- ✗ Only use for lab/dev environments

### Key Takeaway

**For this lab:**
Use Solution 1 (separate RGs). It teaches the real-world pattern you will use in production Azure environments and avoids the policy conflict entirely.

---

**Related concepts:**
- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [Resource group best practices](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/organize-subscriptions)
- [Private endpoint overview](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
