# Azure File Shares: CLI Reference

This file contains Azure CLI (`az`) equivalents for all portal steps in the README.

## Create a file share

```bash
# Create a new file share
az storage share create \
  --account-name storagelab121455 \
  --name share1 \
  --quota 100

# List all file shares in the account
az storage share list \
  --account-name storagelab121455

# Get details about a specific share
az storage share show \
  --account-name storagelab121455 \
  --name share1
```

## Upload files to a file share

```bash
# Upload a single file
az storage file upload \
  --account-name storagelab121455 \
  --share-name share1 \
  --source /path/to/local/file.txt \
  --path file.txt

# Upload an entire directory
az storage file upload-batch \
  --account-name storagelab121455 \
  --share-name share1 \
  --source /path/to/local/directory \
  --destination-path /

# List files in a share
az storage file list \
  --account-name storagelab121455 \
  --share-name share1 \
  --output table

# Download a file
az storage file download \
  --account-name storagelab121455 \
  --share-name share1 \
  --path file.txt \
  --dest /path/to/destination/file.txt
```

## Manage file share properties

```bash
# Increase file share quota (size limit)
az storage share update \
  --account-name storagelab121455 \
  --name share1 \
  --quota 200

# Change access tier (Transaction optimized, Hot, Cool)
az storage share update \
  --account-name storagelab121455 \
  --name share1 \
  --access-tier Cool

# Get share statistics (used space)
az storage share stats \
  --account-name storagelab121455 \
  --name share1
```

## Assign RBAC roles for file shares

```bash
# Assign Storage File Data SMB Share Reader role to a user
az role assignment create \
  --assignee <user-email@example.com> \
  --role "Storage File Data SMB Share Reader" \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-storage-lab/providers/Microsoft.Storage/storageAccounts/storagelab121455

# Assign Storage File Data SMB Share Contributor role
az role assignment create \
  --assignee <user-email@example.com> \
  --role "Storage File Data SMB Share Contributor" \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-storage-lab/providers/Microsoft.Storage/storageAccounts/storagelab121455

# List RBAC role assignments on storage account
az role assignment list \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-storage-lab/providers/Microsoft.Storage/storageAccounts/storagelab121455
```

## Get storage account keys and connection strings

```bash
# List storage account access keys
az storage account keys list \
  --account-name storagelab121455 \
  --resource-group rg-storage-lab

# Get connection string (for scripting)
az storage account show-connection-string \
  --name storagelab121455 \
  --resource-group rg-storage-lab

# Regenerate access keys (for security rotation)
az storage account keys renew \
  --account-name storagelab121455 \
  --resource-group rg-storage-lab \
  --key primary
```

## Mount file shares from command line

### Windows (PowerShell)

```powershell
# Get storage account key
$storageKey = az storage account keys list `
  --account-name storagelab121455 `
  --query '[0].value' -o tsv

# Mount the file share
net use Z: \\storagelab121455.file.core.windows.net\share1 `
  /user:Azure\storagelab121455 $storageKey

# Verify mount
Get-PSDrive Z

# Disconnect mount
net use Z: /delete
```

### Linux (Bash)

```bash
# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name storagelab121455 \
  --query '[0].value' -o tsv)

# Install SMB utilities
sudo apt-get install cifs-utils

# Create mount point
sudo mkdir -p /mnt/azure

# Mount the file share
sudo mount -t cifs \
  //storagelab121455.file.core.windows.net/share1 \
  /mnt/azure \
  -o username=storagelab121455,password=$STORAGE_KEY,vers=3.0

# Verify mount
ls /mnt/azure

# Unmount
sudo umount /mnt/azure
```

## Create and delete file shares (batch operations)

```bash
# Create multiple file shares
for i in {1..5}; do
  az storage share create \
    --account-name storagelab121455 \
    --name share$i \
    --quota 100
done

# Delete a file share
az storage share delete \
  --account-name storagelab121455 \
  --name share1

# Delete all file shares (use with caution!)
az storage share list \
  --account-name storagelab121455 \
  --query '[].name' -o tsv | \
  xargs -I {} az storage share delete \
    --account-name storagelab121455 \
    --name {}
```

## Permissions and ACLs

```bash
# Set NTFS-style permissions on a directory
# Note: ACLs must be set from mounted drive or Azure SDKs
# CLI does not support granular NTFS ACL management directly

# Alternative: Use PowerShell to manage ACLs on mounted drive
# (See PowerShell examples below)
```

### PowerShell: Manage NTFS permissions on mounted file share

```powershell
# After mounting file share as Z: drive

# Get current permissions on a directory
Get-Acl Z:\folder1

# Grant read permission to a user
$acl = Get-Acl Z:\folder1
$permission = New-Object System.Security.AccessControl.FileSystemAccessRule(
  "DOMAIN\Username",
  "Read",
  "ContainerInherit,ObjectInherit",
  "None",
  "Allow"
)
$acl.SetAccessRule($permission)
Set-Acl -Path Z:\folder1 -AclObject $acl

# Grant full control to a group
$acl = Get-Acl Z:\folder1
$permission = New-Object System.Security.AccessControl.FileSystemAccessRule(
  "DOMAIN\GroupName",
  "Modify",
  "ContainerInherit,ObjectInherit",
  "None",
  "Allow"
)
$acl.SetAccessRule($permission)
Set-Acl -Path Z:\folder1 -AclObject $acl
```

---

**Note:** Replace placeholders:
- `storagelab121455` - your storage account name
- `<user-email@example.com>` - actual user email
- `<subscription-id>` - your subscription ID
- `rg-storage-lab` - your resource group name
- `DOMAIN\Username` - domain and username for permissions
