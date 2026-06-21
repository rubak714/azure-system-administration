# 🧯 File Share Access Issues and Troubleshooting Guide

This document details issues encountered during file share creation and access configuration.

## 📍 Overview

Several interdependent factors must align correctly:
- ✓ Authentication methods (RBAC vs storage account keys)
- ✓ Firewall rules and network access
- ✓ Permission propagation timing
- ✓ NTFS-level permissions

---

## ❌ Issue 1: Upload failed - "Not Authorized"

### Error Message
```
Failed to upload 1 out of 1 file(s):
abcd.txt: This request is not authorized to perform this operation.
RequestId:392b767e-201a-0021-7fb0-0180df000000
```

### 🔍 Root Cause
**Firewall blocked access BEFORE identity was checked**

- I had valid RBAC role: **Storage File Data SMB Share Contributor** ✓
- Storage account firewall was set to **Selected networks**
- My IP (`37.111.194.103`) was NOT in the exceptions list ✗
- Firewall rejection happens **before** Azure AD authentication

### ✅ Solution

**Add IP to firewall exceptions:**

1. Storage account → **Networking**
2. Find **Firewalls and virtual networks**
3. Click **+ Add client IP address**
4. Or manually enter IP in CIDR format (e.g., `37.111.194.103/32`)
5. Select **Save**

**Alternative (testing only):**
- Change **Public endpoint connectivity** to **All networks**
- ⚠️ Less secure - revert to "Selected networks" for production

### 💡 Key Insight

**Firewall evaluation order:**
```
Network Layer (Firewall) → ID & RBAC → Data Access
       ↓
    If blocked here, RBAC
    is never even checked!
```

**Why storage key worked:** Likely used alternate authentication path that bypassed initial Entra ID check

---

## ⏳ Issue 2: RBAC Role Not Active - Access Denied

### Error Message
```
Permissions cannot be verified using account with Microsoft Entra ID.

This request is not authorized to perform this operation.
RequestId: b6f32560-601a-000f-7eb2-01d2c8000000
```

### 🔍 Root Cause
**RBAC role assignment has propagation delay**

- I assigned the role: **Storage File Data SMB Share Contributor** ✓
- I tried to authenticate immediately ✗
- Azure AD role changes take **5-15 minutes** to replicate globally
- Storage service hadn't received my updated role yet

### ✅ Solution

**Step 1: Assign the role**
- Storage account → **Access Control (IAM)**
- **+ Add role assignment**
- Role: **Storage File Data SMB Share Contributor** (read/write)
- Members: Selected my user account

**Step 2: Wait for propagation** ⏱️
- **5-15 minutes** minimum
- Check back with Azure AD credentials after waiting

**Step 3: If still failing, use Storage Key as workaround**
- Access account keys as temporary authentication
- Migrate to RBAC once role is active

### 💡 Best Practices
- Assign RBAC roles to **Azure AD groups** (faster propagation than individuals)
- Plan ahead - don't expect immediate access
- Use Storage Key only for emergency/time-critical access

---

## 🔓 Issue 3: Firewall Blocks Azure AD But Not Storage Key

### Symptoms
- Azure AD authentication ✗ fails
- Storage Account Key ✓ works

### Error
```
This storage account's 'Firewalls and virtual networks'
settings may be blocking access. Try adding IP address
('37.111.194.103') to the firewall exceptions.
```

### 🔍 Root Cause
**Different auth paths hit different endpoints**

- Storage account firewall: **Selected networks**
- Azure AD auth attempts: Blocked at firewall layer
- Storage Key auth: May route through alternate path
- **Firewall check happens BEFORE auth method is evaluated**

### ✅ Solution

**Add IP to exceptions:**

1. Storage account → **Networking**
2. **Firewalls and virtual networks**
3. Click **+ Add client IP address** (auto-detect)
4. Or manually enter: `37.111.194.103/32`
5. **Save** (wait a few minutes for rules to apply)

### 🚀 Alternative Approaches

| Approach | Security | Complexity | Use Case |
|----------|----------|-----------|----------|
| Add IP to exception | ⭐⭐⭐ | Low | Recommended for specific IPs |
| All networks | ⭐ | Low | Testing only (not production) |
| Private endpoint | ⭐⭐⭐⭐ | High | Production with VNet access |
| VPN/ExpressRoute | ⭐⭐⭐⭐ | High | On-premises hybrid access |

---

## 🔑 Issue 4: Storage Key vs. RBAC - Which to Use?

### Scenario
I needed file share access across team.  
Decision: Share storage keys or use RBAC roles?

### ❌ DO NOT share storage keys

**Problems with sharing keys:**
- Grant **full access** to entire storage account
- No per-user audit trail (can't see WHO did WHAT)
- If leaked → must rotate key → affects ALL users
- Cannot be selectively revoked
- Encourages credential sharing (security risk)

### ✅ DO assign RBAC roles instead

**Advantages:**
- ✓ Each user has own Azure AD identity
- ✓ Granular roles (reader, contributor, etc.)
- ✓ Full audit trail (WHO, WHAT, WHEN)
- ✓ Revoke without affecting others
- ✓ Works if machine is Azure AD joined

### 📋 Step-by-step: Assign RBAC

```
Storage account → Access Control (IAM)
  ↓
+ Add role assignment
  ↓
Role: Storage File Data SMB Share Contributor
  ↓
Members: Select team member
  ↓
Review + assign
  ↓
⏱️ Wait 5-15 minutes for propagation
  ↓
Team member mounts share:
net use Z: \\storagelab121455.file.core.windows.net\share1
```

### 🤖 For Applications/Scripts

**DO use Azure Key Vault**

❌ Never embed storage key in code  
❌ Never email credentials  
❌ Never commit to git

✅ Store key in Azure Key Vault  
✅ Grant app managed identity access to Key Vault  
✅ Rotate key quarterly automatically

---

## 📛 Issue 5: Net Use Disconnect Failed - "System error 85"

### Error
```
System error 85 has occurred.
The local device name is already in use.
```

### 🔍 Root Cause
**File Explorer or another app still using the drive**

- I tried: `net use Z: /delete`
- Windows blocks disconnect while drive is in use
- File handles still open by running processes

### ✅ Solution

**Step 1: Close all applications**
- Close **all File Explorer windows** (especially Z: drive)
- Close any apps accessing files on Z: drive
- Close text editors, IDEs, any file access

**Step 2: Force disconnect**
```powershell
net use Z: /delete /y
```

**Step 3: If still stuck**
- Restart the machine
- Z: drive automatically unmounts on reboot

### 💡 Prevention
Before disconnecting:
- Use `tasklist /v` to find processes with open files
- Gracefully close applications first
- Then disconnect

---

## 🔄 Issue 6: Key Exposed - Rotate Immediately

### Scenario
Storage key exposed in my terminal output, chat logs, or code.  
What now?

### ✅ IMMEDIATE ACTION: Rotate Key

**Where to rotate:**

1. Storage account → **Access keys**
2. Click **Rotate** (next to exposed key)
3. Confirm rotation
4. Old key becomes invalid **within seconds**

### ⚠️ After Rotation

| System | Impact | Action |
|--------|--------|--------|
| **Storage Browser** | Auto-uses new key | None needed |
| **Mounted Z: drive** | Becomes inaccessible | Remount with new key |
| **Applications** | All connections fail | Update with new key |
| **Scripts** | Need new key update | Refresh Key Vault |

### 🔄 Remount After Key Rotation

```powershell
# Disconnect old mount
net use Z: /delete /y

# Remount with new key
net use Z: \\storagelab121455.file.core.windows.net\share1 `
  /user:Azure\storagelab121455 <NEW-KEY>
```

### 🏭 Production Best Practices

✓ Store keys in **Azure Key Vault**  
✓ Rotate quarterly on schedule  
✓ Use **managed identities** (not keys)  
✓ Audit all key access  
✓ Enable "Storage Account Key Access" only when needed

---

## 📊 Quick Reference: Auth Method Comparison

| Feature | Azure AD Role | Storage Account Key |
|---------|--------------|-------------------|
| **Time to active** | 5-15 minutes ⏱️ | Immediate ⚡ |
| **Security level** | ⭐⭐⭐⭐ High | ⭐⭐ Low |
| **Auditable** | ✓ Full audit trail | ✗ Anonymous access |
| **Best for** | Team members | Emergency only |
| **Can revoke** | ✓ Instantly | ✗ Affects everyone |
| **Propagation delay** | Yes (5-15 min) | No |
| **If compromised** | Revoke role | Rotate key |

---

## 🎯 Summary

**Issues documented:** 6 major authentication & access challenges  
**Root causes:** Firewall, propagation delays, role assignment timing  
**Solutions:** Network exceptions, patience, RBAC best practices  

**Key takeaway:**  
Use **RBAC roles for people**, **Storage Keys for emergencies only**, **Key Vault for apps**.

---

*Documentation of real issues encountered in 02b File Storage lab.  
All issues identified and resolved. See README.md for complete lab guide.*
