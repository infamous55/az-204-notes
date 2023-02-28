---
title: Azure Blob storage
created_at: 12-01-2023
tags: [azure, az-204]
---

## General characteristics of Azure Blob storage

- object storage solution for the cloud, stores unstructured data such as text or binary
- it is the go-to storage solution in Azure, a lot of the other services rely on it
- serves data via HTTP(S) through the REST API, PowerShell, Azure CLI, or client library

## Storage accounts

- provide a unique namespace for your data
- the default endpoint for Blob storage is `http://<mystorageaccount>.blob.core.windows.net/`

### Storage account types

- Standard: general-purpose v2 (supporting Blobs, Queues, Tables, and Azure Files)
  - recommended for most scenarios
  - uses HDDs
- Premium
  - uses SSDs, offers higher performance
  - three types:
    - Premium block blobs
    - Premium page blobs
    - Premium file shares

## Access tiers

- Hot:
  - optimized for frequent access
  - highest storage costs, lowest access costs
- Cool:
  - optimized for infrequently accessed data stored for at least 30 days
  - lower storage costs and higher access costs compared to the Hot tier
- Archive:
  - data is stored offline for at least 180 days
  - high retrieval latency
  - available only for individual block blobs, can't be applied on the container or storage account level
  - only supports LRS, GRS, and RA-GRS
  - lowest storage costs, highest access costs

## Containers

- work like a flat directory inside the storage account
- file names can be used to emulate a nested directory structure

## Blob types

- Block blobs: read from start to finish
- Append blobs: optimized for append operations, ideal for logging
- Page blobs: optimized for random access, ideal for virtual hard drives

## Security features

- all data (including metadata) is encrypted using Storage Service Encryption (SSE)
  - 256-bit AES encryption, FIPS 140-2 compliant
- Azure AD and RBAC are supported for both resource management operations (settings, key management, etc.) and data operations (individual blobs or queues data operations)
- data is transmitted by using Client-Side Encryption, HTTPS, or SMB 3.0
- OS and data disks are encrypted using Azure Disk Encryption (similar to BitLocker)
- delegated access to data objects is possible using a shared access signature (SAS)
- encryption is free, obligatory, and doesn't affect performance
- Microsoft-managed keys are the default, but there are two options for managing encryption with your own keys:
  - specify a customer-managed key that is used to encrypt all data in all services
  - specify a customer-provided key that must be included on the request (REST or client libraries)

## Redundancy options

- LRS (3 local copies)
- ZRS (copies in each availability zone)
- GRS (LRS in the primary region, LRS in the secondary region)
- GZRS (ZRS in the main region, LRS in the secondary region)
- RA-GRS (like GRS with read access)
- RA-GZRS (like GZRS with read access)

Note: the secondary region is always the region pair

## Lifecycle management policies

- used to transition blobs to a cooler storage tier, delete blobs
- based on rules that run once per day at the storage account level
- a policy is a `rules` array in a JSON document (partial JSON is NOT supported)
  - each rule has a `name` (case-sensitive, unique within a policy), `type` (the current valid type is "Lifecycle"), and `definition` object + `enabled` (boolean) option
    - each rule definition includes a filter and an action set
    - filter examples:
      - `blobTypes` (array of predefined values)
      - `prefixMatch`
      - `blobIndexMatch`
    - rule actions:
      - `tierToCool`
      - `enableAutoTierToHotFromCool` (not supported for Archive tier blobs)
      - `tierToArchive`
      - `delete`
    - conditions:
      - `daysAfterModificationGreaterThan`
      - `daysAfterCreationGreaterThan` (used for blob snapshots)
      - `daysAfterLastAccessTimeGreaterThan` (used for a current version of a blob)
      - `daysAfterLastTierChangeGreaterThan` (applies only to `tierToArchive` actions and can be used only with the `daysAfterModificationGreaterThan` condition)

Notes:

- if you define more than one action on the same blob, lifecycle management applies the least expensive action (for example, `delete` is cheaper than `tierToArchive`)
- data stored in a premium block blob storage account cannot be tiered, it must be copied to another account using Put Block URL API or AzCopy

## Rehydration

- rehydrating blob data from the archive tier can be done in two ways:
  - copying the archived blob to an online tier (recommended)
    - the source blob remains unmodified
    - must use different names
  - changing the access tier to an online tier
    - doesn't affect the last modified time, be careful at lifecycle policies that archive blobs based on this condition
- optional `x-ms-rehydrate-priority` header to specify the rehydration priority:
  - Standard (may take up to 15 hours)
  - High (may be complete in under one hour for objects under 10GB in size)

## Container properties and metadata

- the term system properties refers to metadata that already exists on each Blob storage resource and is managed by the Azure Storage client library
- in this context, metadata refers to user-defined metadata
- both properties and metadata are key-value pairs which are valid HTTP headers and C# identifiers
- both properties and metadata are case-insensitive
- partial updates are NOT supported
- setting metadata on a resource overwrites any existing metadata values
- the total size of all metadata pairs can be up to 8KB
- management by using REST:
  - header format: `x-ms-meta-name:string-value`
  - retrieval URI for containers: `GET/HEAD https://myaccount.blob.core.windows.net/mycontainer?restype=container`
  - retrieval URI for blobs: `GET/HEAD https://myaccount.blob.core.windows.net/mycontainer/myblob?comp=metadata`
  - setting URI for containers: `PUT https://myaccount.blob.core.windows.net/mycontainer?comp=metadata&restype=container`
  - setting URI for blobs: `PUT https://myaccount.blob.core.windows.net/mycontainer/myblob?comp=metadata`

Notes:

- standard (system) HTTP headers (properties) for containers include:
  - `ETag`
  - `Last-Modified`
- standard (system) HTTP headers (properties) for blobs include:
  - `ETag`
  - `Last-Modified`
  - `Content-Length`
  - `Content-Type`
  - `Content-MD5`
  - `Content-Encoding`
  - `Content-Language`
  - `Cache-Control`
  - `Origin`
  - `Range`

## Azure Blob storage client library

- .NET classes:
  - `BlobClient`
  - `BlobClientOptions`
  - `BlobContainerClient`
  - `BlobServiceClient`
  - `BlobServiceClient`
- code samples:

  - connect:

  ```csharp
  BlobServiceClient blobServiceClient = new BlobServiceClient(storageConnectionString);
  ```

  - create a container

  ```csharp
  BlobContainerClient containerClient = await blobServiceClient.CreateBlobContainerAsync(containerName);
  ```

  - upload a blob to a container

  ```csharp
  BlobClient blobClient = containerClient.GetBlobClient(fileName);
  using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath)) {
  await download.Content.CopyToAsync(downloadFileStream); downloadFileStream.Close();
  }
  ```

  - list blobs in a container

  ```csharp
  await foreach (BlobItem blobItem in containerClient.GetBlobsAsync()) {
  Console.WriteLine("\t" + blobItem.Name);
  }
  ```

  - download a blob

  ```csharp
  BlobDownloadInfo download = await blobClient.DownloadAsync();
  using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath)) {
  await download.Content.CopyToAsync(downloadFileStream);
  }
  ```

  - delete a container

  ```csharp
  await containerClient.DeleteAsync();
  ```

- properties and metadata management:
  - to retrieve properties use one of the following methods on the `BlobContainerClient` class:
    - `GetProperties`
    - `GetPropertiesAsync`
  - to set metadata use one of the following methods of the `BlobContainerClient` class:
    - `SetMetadata`
    - `SetMetadataAsync`

---
