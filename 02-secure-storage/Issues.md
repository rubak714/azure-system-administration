# 🧯 SAS Token Issues and Troubleshooting Guide

Issues encountered during SAS token configuration and access control.

## 📍 Overview

SAS token generation has **5 interdependent settings** that must align:

1. ✓ Container access level
2. ✓ Storage account key access  
3. ✓ Microsoft Entra authorization
4. ✓ RBAC role assignment
5. ✓ SAS token scope (blob vs container)

If one fails, access fails with cryptic errors.

---

## ❌ Issue 1: Container Access Blocked

### Error
```
Anonymous access to this container is being blocked
```

### 🔍 Root Cause
Container access level is **Disabled** (no public access)  
Even with valid SAS token, Azure blocked my access

### ✅ Solution

1. Storage account → **Containers**
2. Select container (`uploads`)
3. **Change access level**
4. Set to: **Blob** (anonymous read for blobs only)
5. **OK**

### 💡 Key Point
**Container access level is independent from SAS tokens**  
Both must allow access - SAS token + container access level must align

---

## 🔐 Issue 2: Storage Account Key Access Disabled

### Error
```
KeyBasedAuthenticationNotPermitted when trying to generate SAS token
```

### 🔍 Root Cause
- Storage account setting: **"Allow storage account key access"** = OFF  
- SAS token generation requires keys to sign tokens cryptographically
- I couldn't generate SAS tokens without this enabled

### ✅ Solution

1. Storage account → **Configuration**
2. Find: **Allow storage account key access**
3. Toggle: **Enabled**
4. **Save**

### ⚠️ Important
This is different from container access level  
**Both must be enabled** for SAS tokens to work

### 💡 Security Note
Organizations can disable this entirely and use RBAC + managed identity instead  
For this lab, enable it to demonstrate SAS token generation

---

## 👤 Issue 3: User Missing Data-Plane Role

### Error
```
Permissions cannot be verified using account with Microsoft Entra ID
```

### 🔍 Root Cause
I lacked **data-plane RBAC role**

- Subscription **Owner** ✗ does NOT grant blob read access
- Subscription roles are **control-plane** (manage account)
- Blob access requires **data-plane** roles (read/write data)

### ✅ Solution

1. Storage account → **Access Control (IAM)**
2. **+ Add role assignment**
3. Role tab: **Storage Blob Data Reader**
4. Members tab: Select user account
5. **Review + assign**

### 📊 Role Types

| Type | What it controls | Example |
|------|-----------------|---------|
| **Control-plane** | Manage account settings | Owner, Contributor |
| **Data-plane** | Read/write blob data | Storage Blob Data Reader |

**Key insight:** Separate access models - must assign BOTH if needed

---

## 📍 Issue 4: SAS Generated From Container, Not Blob

### Error
```
Value for one of the query parameters specified 
in the request URI is invalid (comp parameter empty)
```

### 🔍 Root Cause
SAS was generated at **container level** instead of **blob level**

- Container-level SAS: Missing blob path ✗
- Blob-level SAS: Includes blob path ✓

### ✅ Correct Process

```
Storage account → Containers → uploads
  ↓
RIGHT-CLICK specific blob (abcd.txt)
  ↓
Generate SAS
  ↓
Permissions: Read ✓
Expiry: 24 hours ✓
Generate SAS URL
  ↓
Copy FULL URL with path + token
```

### 🔍 URL Comparison

**Container-level (❌ WRONG):**
```
https://storage.blob.core.windows.net/uploads?sv=2026...
```

**Blob-level (✅ CORRECT):**
```
https://storage.blob.core.windows.net/uploads/abcd.txt?sv=2026...
```

**The difference:** Blob path must be included in URL

---

## 🖥️ Issue 5: Blank White Page in Browser

### Observation
SAS URL works (no 403) but shows blank page

### ❓ Is This an Error?
**No!** This is expected behavior for plain text files

### ✓ What's Happening
- ✓ SAS token authenticated successfully
- ✓ Azure serving blob content
- ✓ Browser received text file
- ✗ Browser renders plain text as blank page (no formatting)

### ✅ Verification

1. Right-click SAS URL
2. **Save As**
3. Download blob to computer
4. Open locally
5. If content is visible → SAS token works! ✓

### 🎯 To See Content in Browser
- Upload text file with visible content ("Hello World" vs empty `abcd.txt`)
- Upload image or PDF (visual content types)
- Add proper Content-Type headers

**Bottom line:** White page = Success ✓

---

## 🔑 Issue 6: Stored Access Policy SAS Fails

### Error
```
AuthenticationFailed: "Signature did not match. 
String to sign used was /blob/storagelab121455/$root 
contractor-read..."
```

### 🔍 Root Cause
Policy was generated at **container level** (correct)  
BUT SAS token was generated at **container level** (wrong!)

**Both levels required:**
- Policy must exist → container level ✓
- SAS token must be generated → blob level ✓

### ✅ Correct Two-Step Process

**STEP 1: Create policy (container level)**

```
Containers → uploads → Access policy tab
  ↓
+ Add policy
  ↓
Name: contractor-read
Permissions: Read ✓
Expiry: Tomorrow ✓
OK
```

**STEP 2: Generate SAS (blob level)**

```
Containers → uploads → Find blob (abcd.txt)
  ↓
RIGHT-CLICK blob
  ↓
Generate SAS
  ↓
Select policy: contractor-read ✓
Generate SAS URL
  ↓
Copy FULL URL (includes /uploads/abcd.txt)
```

### 💡 Why This Works

| Step | Level | Purpose |
|------|-------|---------|
| **Create policy** | Container | Establish permissions & expiry |
| **Generate SAS** | Blob | Create URL with blob path |

Policy at container + SAS at blob = correct signature ✓

### ❌ Why It Failed Before
Generated SAS at **container level** = URL missing blob path  
Signature validation failed because path didn't match

**Key takeaway:**  
Policy created at **container**, SAS generated from **specific blob**

---

## 📋 Settings Checklist

### All Five Must Be Enabled

| Setting | Location | Value | Lab Status |
|---------|----------|-------|-----------|
| **Container access** | Container → Access level | Blob | ✓ |
| **Account key access** | Config → Key access | Enabled | ✓ |
| **Entra ID in portal** | Config → Auth method | Enabled | ✓ |
| **RBAC role** | IAM → Role assignment | Blob Data Reader | ✓ |
| **SAS scope** | (generated from) | Blob (not container) | ✓ |

### Testing Checklist

Before testing SAS token, verify:

- [ ] Container access level = **Blob**
- [ ] Account key access = **Enabled**
- [ ] Entra ID auth = **Enabled**
- [ ] RBAC role assigned: **Storage Blob Data Reader**
- [ ] SAS generated from **specific blob**
- [ ] Permissions = **Read** (minimum)
- [ ] Expiry = Future date/time
- [ ] URL includes path: `/container/blobname?sv=...`

---

## 🎯 Key Takeaways

✓ Container access level + SAS token both must allow access  
✓ Control-plane roles ≠ Data-plane roles (assign separately)  
✓ SAS must be generated from **blob**, not container  
✓ Blank page = success (browser rendering issue)  
✓ Policy created at container, SAS generated from blob  
✓ All 5 settings must align for access to work

---

*Documentation of issues encountered in 02 Secure Storage lab.  
All issues identified and resolved. See README.md for complete lab guide.*
