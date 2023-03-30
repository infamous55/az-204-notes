---
title: IaaS solutions in Azure
created_at: 20-01-2023
tags: [azure, az-204]
---

## General characteristics of VMs

- use cases:
  - development and testing
  - hosting applications in the cloud
  - extending a datacenter
- the current limit is 20 VMs per region in a single subscription
- Windows VMs have extensions = post deployment automated scripts

## Design considerations

- availability (99.9% SLA for single VM with premium storage, 99.95% SLA for two in the same availability set)
- VM size (or SKU)
- VM images (think Azure Marketplace)
- VM disks
  - types:
    - Standard (HDDs, suitable for development and testing)
    - Premium (SSDs, suitable for production)
  - management options:
    - Managed disks: Microsoft manages everything, it is the recommended disk storage model
    - Unmanaged disks: you manage the storage account that holds your VM disks

## Availability options

- Availability zones (AZs):
  - there are 3 per supported region
  - they are physically and logically separated datacenters with their own independent power source, network, and cooling
  - services can be categorized into:
    - zonal services (resources are pinned to an AZ, for example: VMs, disks, IP addresses)
    - zone-redundant services (resources replicate automatically across AZs, for example: SQL Databases)
- Availability sets:
  - distribute VMs into separate fault domains and update domains
  - a fault domain is a logical group of hardware that share a common power source and network switch = point of failure (think rack)
  - an update domain is a logical group of hardware that can undergo maintenance or be rebooted at the same time (only one update domain is rebooted at a time)
- VM Scale Sets
  - a scale set is a group of load balanced VMs
  - the VMs connect to the load balancer using their NICs
  - you define a front-end IP configuration that contains one or more public IP addresses which allows your load balancer and applications to be accessible over Internet
  - Azure Load Balancer works at Layer 4 (TCP, UDP)
  - you can define load balancer rules for specific ports and protocols that map to your VMs

## VM types

- General Purpose: for testing and development, small to medium databases, and low to medium traffic web servers
- Compute Optimized: for medium traffic web servers, network appliances, batch processes, and application servers
- Memory Optimized: for relational database servers, medium to large caches, and in-memory analytics
- Storage Optimized: for Big Data, SQL/NoSQL databases, data warehousing and large transactional databases
- GPU: for heavy graphic rendering, video editing, and model training and inferencing with deep learning
- High Performance Compute: fastest and most powerful CPUs with optional high-throughput network interfaces

## General characteristics of ARM templates

- JSON files
- IaC benefits (declarative syntax, repeatable results, automatic ordering of operations and parallelization when possible)
- templates are converted into REST API operations
- a template has the following sections:
  - parameters
  - variables (to define reusable values)
  - user-defined functions
  - resources
  - outputs (returns values from the deployed resources)
- if the resources specified already exists, ARM performs an updated instead of creating a new asset
- extensions
- VS Code integration and nice autocomplete
- can be shared through Template specs (template storage in the cloud, integrates with RBAC)

## Conditional deployment

- a resource is deployed only if the `condition` is true
- conditional deployment doesn't cascade to child resources
- if you use a `reference` or `list` function with a resource that is conditionally deployed, the function is evaluated even if the resource isn't deployed
- newOrExisting parameter example:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "newOrExisting": {
      "type": "string",
      "defaultValue": "new",
      "allowedValues": ["new", "existing"]
    }
  },
  "functions": [],
  "resources": [
    {
      "condition": "[equals(parameters('newOrExisting'), 'new')]",
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-06-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS",
        "tier": "Standard"
      },
      "kind": "StorageV2",
      "properties": {
        "accessTier": "Hot"
      }
    }
  ]
}
```

## Deployment models

- Complete mode: ARM deletes the resources that exist in the resource group and aren't specified in the template
- Incremental mode: ARM leaves unchanged the resources that exist in the resource group but aren't specified in the template
- CLI example:

```bash
az deployment group create \
  --mode Complete \
  --name ExampleDeployment \
  --resource-group ExampleResourceGroup \
  --template-file storage.json
```

## General characteristics of Azure Container Registry (ACR)

- Docker Hub idea in the cloud
- there are three service tiers:
  - Basic: lower included storage and image throughput, optimized for development
  - Standard: increased included storage and image throughput, ideal for production scenarios
  - Premium: highest amount of included storage, image throughput, and concurrent operations + premium features (geo-replication, content trust for image tag signing, and private link with private endpoints)
- each image is a read-only snapshot of a Docker-compatible container
- supports both Linux and Windows images
- can store other formats (Helm charts, OCI)

## Storage capabilities

- Encryption-at-rest
- Regional storage
- ZRS (only in the Premium service tier)
- Scalable storage

## ACR tasks

- enable automated builds
- task scenarios:
  - quick task: manually running `az acr build`, equivalent to `docker build` and `docker push` in the cloud
  - automatically triggered tasks (on source code update, base image update, or on a schedule)
  - multi-step task: defined in a YAML file, can be used with multiple containers

## Dockerfile example

```dockerfile
# Step 1: Specify the parent image for the new image
FROM ubuntu:18.04

# Step 2: Update OS packages and install additional software
RUN apt -y update &&  apt install -y wget nginx software-properties-common apt-transport-https \
  && wget -q https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb \
  && dpkg -i packages-microsoft-prod.deb \
  && add-apt-repository universe \
  && apt -y update \
  && apt install -y dotnet-sdk-3.0

# Step 3: Configure Nginx environment
CMD service nginx start

# Step 4: Configure Nginx environment
COPY ./default /etc/nginx/sites-available/default

# STEP 5: Configure work directory
WORKDIR /app

# STEP 6: Copy website code to container
COPY ./website/. .

# STEP 7: Configure network requirements
EXPOSE 80:8080

# STEP 8: Define the entry point of the process that runs in the container
ENTRYPOINT ["dotnet", "website.dll"]
```

## General characteristics of Azure Container Instances (ACI)

- container deployment solution in the cloud, works with a small number of containers organized in container groups (doesn't have full orchestration capabilities like Azure Kubernetes Service)

## Container groups

- container groups are collections of containers with the same lifecycle, local network, and storage volumes
- container groups are exposed to the internet with a public IP and a fully qualified domain name
  - all containers in a group share the port namespace
- the containers are isolated, and can have custom sizes (similar to VMs)
- both Linux and Windows containers are available, but multi-container groups work only with Linux
- you can mount Azure Files shares to a container to persist state
- container groups can be deployed by using an ARM Template (when you need to deploy additional resources) or YAML files (when you need to deploy only container instances)

### Scenario examples

- an application container and a continuous deployment container
- an application container and a logging container
- an application container and a monitoring container
- a front-end container and a backend container

### CLI examples

- create/start a container instance

```bash
DNS_NAME_LABEL=aci-example-$RANDOM

az container create --resource-group az204-aci-rg \
  --name myContainer \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --ports 80 \
  --dns-name-label $DNS_NAME_LABEL \
  --location myLocation
```

- verify the container is running

```bash
az container show --resource-group az204-aci-rg \
  --name mycontainer \
  --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
  --out table
```

## Restart policies

- `Always`: containers in the container group are always restarted, this is the default setting
- `Never`: containers in the container group are never restarted, they run at most once
- `OnFailure`: containers in the container group are restarted only when the process executed terminates with a nonzero exit code (error), so they run at least once
- the restart policy is specified in the `--restart-policy` parameter when you call `az container create`
- the state of a stopped container whose restart policy is `Never` or `OnFailure` is set to **Terminated**

## Environment variables

- visible in the container's properties (not secure) by using the `--environment-variables` flag at creation
- secure values, accessible only from within the container, can be supplied in the YAML file

### Secure values example

```yaml
apiVersion: 2018-10-01
location: eastus
name: securetest
properties:
  containers:
    - name: mycontainer
      properties:
        environmentVariables:
          - name: 'NOTSECRET'
            value: 'my-exposed-value'
          - name: 'SECRET'
            secureValue: 'my-secret-value'
        image: nginx
        ports: []
        resources:
          requests:
            cpu: 1.0
            memoryInGB: 1.5
  osType: Linux
  restartPolicy: Always
tags: null
type: Microsoft.ContainerInstance/containerGroups
```

```bash
az container create --resource-group myResourceGroup \
  --file secure-env.yaml
```

## Mounting volumes to container instances

- for the purpose of persisting state (because containers are stateless by default)
- using Azure Files shares and the SMB protocol
- only works with Linux containers running as _root_

### Examples

- using the CLI

```bash
az container create \
    --resource-group $ACI_PERS_RESOURCE_GROUP \
    --name hellofiles \
    --image mcr.microsoft.com/azuredocs/aci-hellofiles \
    --dns-name-label aci-demo \
    --ports 80 \
    --azure-file-volume-account-name $ACI_PERS_STORAGE_ACCOUNT_NAME \
    --azure-file-volume-account-key $STORAGE_KEY \
    --azure-file-volume-share-name $ACI_PERS_SHARE_NAME \
    --azure-file-volume-mount-path /aci/logs/
```

- using an YAML file

```yaml
apiVersion: '2019-12-01'
location: eastus
name: file-share-demo
properties:
  containers:
    - name: hellofiles
      properties:
        environmentVariables: []
        image: mcr.microsoft.com/azuredocs/aci-hellofiles
        ports:
          - port: 80
        resources:
          requests:
            cpu: 1.0
            memoryInGB: 1.5
        volumeMounts:
          - mountPath: /aci/logs/
            name: filesharevolume
  osType: Linux
  restartPolicy: Always
  ipAddress:
    type: Public
    ports:
      - port: 80
    dnsNameLabel: aci-demo
  volumes:
    - name: filesharevolume
      azureFile:
        sharename: acishare
        storageAccountName: storageaccountname
        storageAccountKey: storageaccountkey
tags: {}
type: Microsoft.ContainerInstance/containerGroups
```

- using an ARM template (JSON file) for multiple volumes

```JSON
...
"volumeMounts": [{
  "name": "myvolume1",
  "mountPath": "/mnt/share1/"
},
{
  "name": "myvolume2",
  "mountPath": "/mnt/share2/"
}]
...
```
