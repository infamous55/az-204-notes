---
title: Azure Key Vault, Managed Identities, and App Configuration
created_at: 13-02-2023
tags: [azure, az-204]
---

## General characteristics of Azure Key Vault

- is used for managing secrets (tokens, passwords, API keys, etc.), keys (RSA and Elliptic Curve Keys), and certificates
- supports two types of containers:
  - vaults, which can store both software and HSM-backed keys, secrets, and certificates
  - HSM pools, which can only store HSM-backed key
- has two service tiers:
  - Standard, which encrypts with a software key
  - Premium, which includes HSM-protected keys
- data is always encrypted, even in transit
- it integrates with your other applications running in Azure
- authentication is done via AAD, and authorization may be done via Azure RBAC or Key Vault access policies
- there is an option to log all the activity and send that information to a storage account, an event hub, or Azure Monitor logs

## Best practices

- use **managed identities** for when other Azure resources need access to your Key Vault
- use **separate Key Vaults** for different environments (Development, Production, etc.)
- create regular **backups**
- be sure to turn on **logging and alerts**
- turn on **soft-delete** (such that when data is deleted, it transitions to a state of soft deleted instead of being permanently erased) and **purge protection** (when purge protection is on, a vault or an object in the deleted state cannot be purged until the retention period has passed)

## General characteristics of managed identities

- they provide an identity for applications to use when connecting to resources that support Azure Active Directory authentication
- there are two types of managed identities:
  - system-assigned managed identity: is enabled directly on an Azure service instance, and shares its lifecycle (if that instance is deleted, Azure automatically cleans up the credentials and the identity)
  - user-assigned managed identity: is created as a standalone Azure resource, can be assigned to one or more Azure service instances, has its separate lifecycle
- internally, managed identities are service principals which are locked to only be used with Azure resources

## Configuration example for virtual machines

- to enable a system-assigned managed identity for a virtual machine, your account needs the Virtual Machine Contributor role
- to assign a user-assigned identity to a virtual machine, your account needs the Virtual Machine Contributor and Managed Identity Operator role assignments

### CLI examples

- enable system-assigned managed identity on a virtual machine during the creation

```bash
az vm create --resource-group myResourceGroup \
  --name myVM --image win2016datacenter \
  --generate-ssh-keys \
  --assign-identity \
  --role contributor \
  --scope mySubscription \
  --admin-username azureuser \
  --admin-password myPassword12
```

- enable system-assigned managed identity on an existing virtual machine

```bash
az vm identity assign -g myResourceGroup -n myVm
```

- create a user-assigned managed identity

```bash
az identity create -g myResourceGroup -n myUserAssignedIdentity
```

- assign a user-assigned managed identity during the creation of a virtual machine

```bash
az vm create \
  --resource-group <RESOURCE_GROUP> \
  --name <VM_NAME> \
  --image UbuntuLTS \
  --admin-username <USER_NAME> \
  --admin-password <PASSWORD> \
  --assign-identity <USER_ASSIGNED_IDENTITY_NAME> \
  --role <ROLE> \
  --scope <SUBSCRIPTION>
```

- assign a user-assigned managed identity to an existing virtual machine

```bash
az vm identity assign \
  -g <RESOURCE_GROUP> \
  -n <VM_NAME> \
  --identities <USER_ASSIGNED_IDENTITY>
```

## Acquiring an access token

- a client application can request an access token for accessing a given resource by using the managed identity service principal
- in the case of using a managed identity for virtual machines, the client uses an Azure IMDS (Azure Instance Metadata Service) endpoint inside the virtual machine instead of an AAD endpoint
- request example: `GET 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' HTTP/1.1 Metadata: true`
- it is recommended to cache tokens in your code

## General characteristics of Azure App Configuration

- is a fully managed service that complements Azure Key Vault, and it is used to:
  - centralize management and distribution of hierarchical configuration data in a secure manner
  - dynamically change application settings
  - control feature availability in real-time (using feature flags, as described in [[Azure App Service]])

## Configuration data as key-value pairs

- when used inside key names, special characters need to be escaped
- flat vs hierarchical namespaces (using delimiters such as `/` or `:`)
- keys have labels to differentiate values with the same key (or uniquely identify each key), the default label is `null`
- App Configuration doesn't version key values automatically as they're modified, but you can achieve this by using labels
- values are unicode strings

## Secure app configuration

- you can choose to use customer-managed keys for the encryption
  - this option is only available in the Standard tier of Azure App Configuration, you also need to have a Key Vault with soft-delete and purge-protection features enabled, and an RSA or RSA-HSM key that meets the requirements within your key vault
- you can use private endpoints for Azure App Configuration to allow clients on a VNet to securely access data
  - the private endpoint uses an IP address from the VNet address space for your App Configuration store; this way, the network traffic traverses over the VNet using a private link which eliminates exposure to the public internet
  - when using a private endpoint, the DNS resource record is changed to an alias in a subdomain with the prefix `privatelink` resolving only from within the VNet

---
