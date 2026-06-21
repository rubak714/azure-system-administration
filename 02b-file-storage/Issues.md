# 🧯 File Share Access Issues and Troubleshooting Guide

This document details all issues encountered during file share creation and access configuration, and how to fix them.

## Overview

During the file share setup and access configuration process, several interdependent factors must align correctly. Authentication methods (RBAC vs storage account keys), firewall rules, and permission propagation timing caused multiple issues. Here's what went wrong and how each issue was fixed:

---

## Issue 1: File share upload failed with "This request is not authorized to perform this operation"

**Error:** 
```
Failed to upload 1 out of 1 file(s):
abcd.txt: This request is not authorized to perform this operation.
RequestId:392b767e-201a-0021-7fb0-0180df000000
```

**Root cause:**
- User attempted to upload via Storage Browser using Azure AD (Entra ID) credentials
- While the user had assigned RBAC role (**Storage File Data SMB Share Contributor**), the upload still failed
- The actual issue was a **firewall rule blocking access** before the RBAC role was even evaluated
- Storage account firewall was set to **Selected networks**, and user's IP was not in the exception list

**What was changed:**
- Added user's IP address (`37.111.194.103`) to the firewall exceptions
- User then switched from Azure AD authentication to **Storage Account Key** authentication in Storage Browser

**Where to fix:**
1. Open the storage account
2. Select **Networking** from the left menu
3. Look for **Firewalls and virtual networks**
4. If set to **Selected networks**, add your client IP:
   - Click **+ Add your client IP address**
   - Or manually enter your IP in the allowed range
5. Select **Save**
6. Alternatively, change **Public endpoint connectivity** to **All networks** (less secure, only for testing)

**Why this happened:**
- Azure Storage firewall is evaluated **before** identity and RBAC roles
- If the firewall blocks your IP, authentication is never checked
- The error message said "not authorized," but the real issue was network access, not permissions

**Why Storage Account Key worked:**
- Storage Account Key uses a different authentication path that was properly configured
- The firewall exception + storage key combination bypassed the Entra ID propagation delay

---

## Issue 2: Azure AD RBAC role assignment didn't grant immediate access

**Error:** 
```
You do not have permissions to list the data using your user account with Microsoft Entra ID. 
Click to learn more about authenticating with Microsoft Entra ID. 
This request is not authorized to perform this operation using this permission.
RequestId:b6f32560-601a-000f-7eb2-01d2c8000000
Time:2026-06-21T19:15:32.8409115Z
```

**Root cause:**
- User assigned **Storage File Data SMB Share Contributor** RBAC role to their account
- Immediately tried to authenticate via Storage Browser using Azure AD credentials
- RBAC role assignment has a **propagation delay** of 5-15 minutes across Azure services
- Storage Browser checked permissions before the role was fully replicated to the Storage service

**What was changed:**
- Waited 5-15 minutes for RBAC role propagation to complete
- User switched to **Storage Account Key** authentication as a workaround (bypasses Entra ID entirely)

**Where to fix:**
1. Open the storage account
2. Select **Access Control (IAM)**
3. Select **+ Add role assignment**
4. Choose role: **Storage File Data SMB Share Contributor** (for read/write file share access)
5. Assign to: Your user account
6. Select **Review + assign**
7. **Wait 5-15 minutes** before attempting authentication with Azure AD credentials
8. If timeout continues, use Storage Account Key as temporary workaround

**Why this is needed:**
- RBAC assignments are distributed across Azure's global infrastructure
- Storage Blob Data roles must sync to the Storage service before the role takes effect
- During this propagation window, authentication attempts will fail

**Best practice:**
- For production, assign RBAC roles to **Azure AD groups** (not individual users) to reduce propagation issues
- Plan ahead and assign roles before users need immediate access
- Use Storage Account Key for critical time-sensitive operations, then migrate to RBAC once role is active

---

## Issue 3: Firewall blocking Azure AD authentication but not storage key

**Error:** 
```
This storage account's 'Firewalls and virtual networks' settings may be blocking access 
to storage services. Try adding your client IP address ('37.111.194.103') to the firewall 
exceptions, or by allowing access from 'all networks' instead of 'selected networks'.
```

**Root cause:**
- Storage account firewall was set to **Selected networks** with no IP exceptions
- Different authentication methods route through different network endpoints
- Azure AD authentication to Storage Browser was blocked by firewall **before** RBAC role was checked
- Storage Account Key might be using a different authentication path that was partially accessible

**What was changed:**
- Added user's client IP (`37.111.194.103`) to firewall exceptions
- Changed authentication method from **Azure AD** to **Storage Account Key** in Storage Browser

**Where to fix:**
1. Open the storage account
2. Select **Networking** from the left menu
3. Check **Firewalls and virtual networks**
4. If **Public endpoint connectivity** is set to **Selected networks**:
   - Click **+ Add your client IP address**
   - This automatically detects and adds your current IP
   - Or manually enter your IP in CIDR format (e.g., `37.111.194.103/32`)
5. Select **Save**
6. Wait a few minutes for firewall rules to apply

**Alternative approach (for testing only):**
1. Change **Public endpoint connectivity** to **All networks**
2. This disables the firewall entirely (less secure)
3. Only for testing; revert to "Selected networks" with explicit IPs for production

**Why this happened:**
- Firewall rules are applied at the network ingress layer, independent of authentication methods
- Both Azure AD and Storage Key authentication must pass the firewall check first
- The error message appearing after adding the IP suggests the firewall had already blocked the request

---

## Issue 4: Storage Account Key vs. RBAC - Which to use for team members?

**Scenario:**
- Owner wants to grant file share access to team members
- Uncertain whether to share storage account keys or use RBAC roles

**Decision made:**

**❌ DO NOT share storage account keys**
- Keys grant **full access** to all data in the entire storage account
- No per-user auditing (cannot see who did what, only "someone with the key")
- If leaked, must rotate key (affects all users immediately)
- Cannot be revoked selectively

**✅ DO assign RBAC roles instead**
- Each user authenticates with their own Azure AD identity
- Role-based access (e.g., read-only, read/write)
- Fully auditable (can see exactly who accessed what, when)
- Can be revoked immediately without affecting others
- Works if user's machine is Azure AD joined

**Where to assign:**
1. Open the storage account
2. Select **Access Control (IAM)**
3. Select **+ Add role assignment**
4. Choose role: **Storage File Data SMB Share Contributor** (read/write) or **Storage File Data SMB Share Reader** (read-only)
5. Assign to: Team member's user account or Azure AD group
6. Select **Review + assign**
7. Team member waits 5-15 minutes for role to propagate
8. Team member runs: `net use Z: \\storagelab121455.file.core.windows.net\share1` (no key needed)

**For applications/scripts (not people):**
- Store storage account key in **Azure Key Vault** (not email/Slack)
- Grant only the application access to the Key Vault
- Rotate key regularly (e.g., quarterly)

---

## Issue 5: Net use command returned "System error 85" when disconnecting

**Error:**
```
System error 85 has occurred.
The local device name is already in use.
```

**Root cause:**
- Attempted to disconnect mounted Z: drive with `net use Z: /delete`
- File Explorer window or another application still had Z: drive open
- Cannot disconnect a drive that is actively in use

**What was changed:**
- Closed all File Explorer windows accessing Z: drive
- Ran command with force flag: `net use Z: /delete /y`

**Where to fix:**
1. Close **all File Explorer windows** (especially those showing Z: drive)
2. Close **all applications** that might be accessing files on Z: drive
3. Run the disconnect command:
   ```powershell
   net use Z: /delete /y
   ```
4. If still stuck, restart the machine (Z: drive disconnects on reboot automatically)

**Why this is needed:**
- Windows locks a mounted drive if any process has an open file handle
- The `/y` flag forces disconnection (bypasses the "are you sure" prompt)
- Restarting ensures all file handles are released

---

## Issue 6: Credential rotated after exposure - What to do next

**Scenario:**
- Storage account key was exposed in terminal commands and chat logs
- Key was rotated immediately via Azure portal
- User needed to know next steps

**Decision made:**

**✅ Regenerate storage account key immediately after exposure**

**Where to rotate:**
1. Open the storage account
2. Select **Access keys** from the left menu
3. Click **Rotate** next to the exposed key (Key 1 or Key 2)
4. Confirm the rotation
5. The old key becomes invalid within seconds
6. Any systems using the old key will fail (this is intentional)

**After rotation:**
- **Storage Browser:** Will automatically use the new key (no action needed)
- **Mounted Z: drive:** Will become inaccessible; must remount with new key:
  ```powershell
  net use Z: /delete /y
  net use Z: \\storagelab121455.file.core.windows.net\share1 /user:Azure\storagelab121455 <NEW-KEY>
  ```
- **Applications/scripts:** Must be updated with the new key (or use Azure Key Vault to manage rotation automatically)

**Best practice for production:**
- Store keys in **Azure Key Vault** (not config files or command line)
- Rotate keys **quarterly** on a schedule
- Use **managed identities** for applications instead of storage keys
- Enable **Storage Account Key Access** toggle only when necessary

---

## Summary: Azure AD vs Storage Key Authentication

| Aspect | Azure AD (RBAC Role) | Storage Account Key |
|--------|---------------------|-------------------|
| **Setup time** | Role assignment + 5-15 min propagation | Immediate (copy key) |
| **Security** | Per-user identity, auditable, revocable | Full account access, not auditable |
| **Best for** | Team members, employees | Emergency access, applications in Key Vault |
| **Firewall interaction** | Blocks if firewall prevents Entra ID access | May work via alternate endpoint |
| **Propagation delay** | 5-15 minutes | None |
| **Audit trail** | Yes (every action logged with user identity) | No (just "someone with the key") |
| **If compromised** | Revoke role immediately (others unaffected) | Rotate key (affects all users) |

---

*Documentation of issues encountered in 02b File Storage lab. All issues have been identified and resolved. See README.md for the complete lab guide.*
