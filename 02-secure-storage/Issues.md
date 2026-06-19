# 🧯 SAS Token Issues and Troubleshooting Guide

This document details all issues encountered during SAS token configuration and how to fix them.

## Overview

During the SAS token generation process, several interdependent settings must be configured correctly. Here's what went wrong and how each issue was fixed:

---

## Issue 1: Container access level was disabled

**Error:** "Anonymous access to this container is being blocked"

**Root cause:**
- The container access level was set to **Disabled** (no public access)
- Even with a valid SAS token, Azure blocks access if the container forbids it

**What was changed:**
- Container access level: **Disabled** → **Blob** (anonymous read access for blobs only)

**Where to fix:**
1. Open the storage account
2. Select **Containers** > select the container (`uploads`)
3. Select **Change access level**
4. Set **Anonymous access level** to **Blob (anonymous read access for blobs only)**
5. Select **OK**

**Why this is needed:**
- The container needs to allow blob-level access, even when you're using SAS tokens
- This setting works independently from SAS tokens—both must allow access

---

## Issue 2: Storage account key access was disabled

**Error:** "KeyBasedAuthenticationNotPermitted" when trying to generate SAS token

**Root cause:**
- The storage account had "Allow storage account key access" disabled
- SAS token generation requires access to storage account keys to sign the token
- Without this enabled, the portal cannot generate a valid SAS

**What was changed:**
- Storage account key access: **Disabled** → **Enabled**

**Where to fix:**
1. Open the storage account
2. Select **Configuration** from the left menu
3. Look for **Allow storage account key access**
4. Set the toggle to **Enabled**
5. Select **Save**

**Why this is needed:**
- The storage account key is used cryptographically to sign and validate SAS tokens
- This is a deliberate security control—organizations can disable keys if they use only RBAC and managed identity
- For this lab, we need it enabled to demonstrate SAS token generation

**Note:** This is different from the container access level. Both must be enabled for SAS tokens to work.

---

## Issue 3: User lacked Storage Blob Data Reader RBAC role

**Error:** "You do not have permissions to list the data using your user account with Microsoft Entra ID"

**Root cause:**
- The user had no **data-plane** RBAC role
- Control-plane roles (Owner, Contributor) do NOT grant blob read access
- Data-plane roles (Storage Blob Data Reader, Storage Blob Data Contributor) are required separately

**What was changed:**
- Assigned **Storage Blob Data Reader** RBAC role to the user account

**Where to fix:**
1. Open the storage account
2. Select **Access Control (IAM)** from the left menu
3. Select **+ Add** > **Add role assignment**
4. On the **Role** tab, search for and select **Storage Blob Data Reader**
5. On the **Members** tab, select **+ Select members**
6. Find and select your user account
7. Select **Review + assign**

**Why this is needed:**
- Azure Storage separates **control-plane** (managing the account) from **data-plane** (reading blob data)
- Even if you're the Owner of the subscription, you cannot read blobs without a data-plane role
- Storage Blob Data Reader is the appropriate role for read-only access

**Why Microsoft Entra authorization in the portal must be enabled:**
- The portal uses your Azure AD identity to check your RBAC roles
- Without "Default to Microsoft Entra authorization in the Azure portal" enabled, the portal cannot verify that you have Storage Blob Data Reader
- This is a security feature: the portal defaults to using your signed-in user credentials, not account keys

---

## Issue 4: SAS token generated from container instead of blob

**Error:** "Value for one of the query parameters specified in the request URI is invalid (comp parameter empty)"

**Root cause:**
- The SAS token was generated from the **container level** instead of the **blob level**
- Container-level SAS tokens do not include the specific blob path in the URL
- The Azure Storage service cannot identify which blob to serve

**What was changed:**
- Regenerated SAS token directly from the **specific blob** (not the container)

**Correct process:**
1. Open the storage account
2. Select **Containers** > select the container (`uploads`)
3. **Right-click the specific blob** (`abcd.txt`) 
4. Select **Generate SAS**
5. Set **Permissions** to read-only
6. Set **Expiry** (e.g., 24 hours from now)
7. Select **Generate SAS URL and token**
8. Copy the **SAS URL** (the full URL including path and token)

**Why this is needed:**
- Blob-level SAS tokens produce a URL like: `https://storagelab121455.blob.core.windows.net/uploads/abcd.txt?sv=2026-02-06&...`
- Container-level SAS tokens produce: `https://storagelab121455.blob.core.windows.net/uploads?sv=2026-02-06&...` (missing the blob name)
- The full path (including `/uploads/abcd.txt`) tells Azure Storage exactly which blob to return

**Key difference:**

| Level | URL Pattern | Use case |
|-------|-------------|----------|
| Container | `/uploads?sv=2026...` | Listing all blobs in container |
| Blob | `/uploads/abcd.txt?sv=2026...` | Accessing a specific blob ✓ Correct for read access |

---

## Issue 5: White/blank page when accessing blob via SAS URL

**Observation:** SAS URL works (no 403 Forbidden error) but shows a blank white page in the browser

**Is this an error?**
- **No.** This is expected behavior for plain text files.

**What's happening:**
- ✓ The SAS token authenticated successfully
- ✓ Azure is serving the blob content
- ✓ The browser receives the text file
- The browser renders plain text as a blank page (no HTML formatting)

**Verification that it works:**
1. Right-click the SAS URL in the browser
2. Select **Save As**
3. Download the blob to your computer
4. Open the downloaded file locally
5. If you see the content, the SAS token is fully functional

**To see content displayed in the browser:**
- Upload a text file with visible content (e.g., "Hello World") instead of `abcd.txt`
- Or upload an image or PDF (visible content types)
- Or add Content-Type headers to serve as `text/html` or `text/plain with formatting`

**Summary for this issue:**
White page = ✅ Success. The SAS token is working. The blank appearance is just how browsers render plain text files without formatting.

---

## Summary table: All settings that must be enabled

| Setting | Location | Default | Lab Setting | Why it matters |
|---------|----------|---------|-------------|----------------|
| **Container access level** | Container > Access level | Disabled | Blob | Controls who can access blobs in the container |
| **Storage account key access** | Storage account > Configuration | Enabled (typical) | Enabled | Required to generate SAS tokens |
| **Microsoft Entra in portal** | Storage account > Configuration | Enabled (typical) | Enabled | Portal uses your Azure AD identity to check RBAC roles |
| **Storage Blob Data Reader role** | Storage account > IAM | (None by default) | Assigned to user | Grants data-plane permission to read blobs |
| **SAS token scope** | (N/A) | (N/A) | Blob level | Token must point to specific blob, not just container |

---

## Why these settings are interdependent

All five settings must align for SAS token access to work:

1. **Container must allow blob access** (access level = Blob)
2. **Account must allow key-based SAS** (key access = Enabled)
3. **Portal must know your RBAC role** (Entra auth = Enabled)
4. **User must have blob read permission** (RBAC = Storage Blob Data Reader)
5. **SAS token must point to the blob** (not just container)

If even one is misconfigured, access fails with a cryptic error message. This is why troubleshooting SAS issues requires checking all five points.

---

## Quick reference checklist

Before testing a SAS token, verify all of these are configured:

- [ ] Container access level = **Blob** (or appropriate public access level)
- [ ] Storage account key access = **Enabled**
- [ ] Microsoft Entra authorization in portal = **Enabled**
- [ ] User has **Storage Blob Data Reader** RBAC role assigned
- [ ] SAS token generated from **specific blob**, not container
- [ ] SAS token has **read** permission (at minimum)
- [ ] SAS token expiry is set to future date/time
- [ ] URL includes full path: `/container/blobname?sv=...`
