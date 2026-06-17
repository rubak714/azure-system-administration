# CLI cross-reference (optional)

You do not need this to finish the lab - it is all done in the portal.
But the exam shows you `az` commands and asks what they do, so skim these
and match each one to the portal step you performed.

```bash
# OPTIONAL cross-check only. The lab itself is done in the Azure portal.
# These are the same actions in Azure CLI so you recognise them on the exam.

# Sign in and set the subscription
az login
az account set --subscription "<your-subscription-id>"

# Create a resource group
az group create --name rg-identity-lab --location eastus

# Create a test user (Entra ID)
az ad user create --display-name "Lab Tester" \
  --user-principal-name labtester@<yourtenant>.onmicrosoft.com \
  --password "<StrongPassw0rd!>"

# Assign the Reader role to a group at resource-group scope
az role assignment create --assignee "<group-object-id>" \
  --role "Reader" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-identity-lab

# Assign the 'Allowed locations' built-in policy
az policy assignment create --name "allowed-locations" \
  --policy "e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --params '{ "listOfAllowedLocations": { "value": ["eastus"] } }' \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-identity-lab

# Put a ReadOnly lock on a resource group
az lock create --name protect-rg --lock-type ReadOnly \
  --resource-group rg-identity-lab

```
