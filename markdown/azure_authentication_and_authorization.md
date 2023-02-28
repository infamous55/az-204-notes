---
title: Azure Authentication and Authorization
created_at: 03-02-2023
tags: [azure, az-204]
---

## General characteristics of the Microsoft identity platform

- used to sign in users with their Microsoft identities or social accounts
- provides authorized access to your own APIs or Microsoft APIs like Microsoft Graph
- components:
  - OAuth 2.0 and OpenID Connect standard-compliant authentication service
  - Open-source libraries (MSAL and others)
  - Application management portal
  - Application configuration through the Microsoft Graph API and PowerShell

## Application objects and Service principals

- an application registered with Azure Active Directory can be Single tenant or Multi-tenant
- the registration automatically creates an application object and a service principal in the home tenant
- the application object acts like a global class from which one or more local service principal objects can be created
- the application object describes:
  - how the service can issue tokens in order to access the application
  - resources that the application might need to access
  - actions that the application can take
- each security principal represents an entity (user or service) inside a tenant, and it is used to define the access policy and permissions for that entity inside that specific AAD tenant
- types of service principal:
  - Application (works as described above)
  - Managed identity
  - Legacy (for legacy apps, Single tenant only)

## Permissions and consent

- can ask for permissions in the `scope` query parameter when logging in with OAuth 2.0
- each permission is indicated by appending the permission value to the resource's identifier or application ID URI like in `https://graph.microsoft.com/Calendars.Read`
  - if the resource identifier is omitted, the resource is assumed to be Microsoft Graph
- permission types:
  - Delegated permissions: the app is delegated with the permission to act as a signed-in user when it makes calls
  - Application permissions: used by apps that run without a signed-in user, only administrators can consent to these permissions
- consent types:
  - static user consent: all required permissions are specified in the app's configuration and are requested when the user first signs-in
  - "incremental and dynamic" user consent: asks for a minimum set of permissions upfront and requests more over time, when needed
  - admin consent: for when your app needs access to certain high-privilege permissions
- admins can consent on behalf of an organization (all the users in a tenant) when the permissions are static
- authorization request example:

```GET <https://login.microsoftonline.com/common/oauth2/v2.0/authorize>?
client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=code
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&response_mode=query
&scope=
https%3A%2F%2Fgraph.microsoft.com%2Fcalendars.read
%20
https%3A%2F%2Fgraph.microsoft.com%2Fmail.send
&state=12345
```

## Conditional access

- feature in Azure Active Directory
- used for:
  - multifactor authentication
  - allowing only specific devices to access specific services
  - restricting user locations and IP ranges
- the code needs to be modified when using conditional access in these situations:
  - performing the on-behalf-of flow
  - accessing multiple services/resources
  - SPAs using MSAL.js
  - web apps calling a resource

## General characteristics of Microsoft Authentication Library

- can be used to provide secure access to Microsoft APIs, third-party APIs, or your own API
- supports many different architectures and programming languages
- handles things like token acquisition, caching, and refreshing
- applications using MSAL are separated into two categories:
  - public client applications (they run locally or in a browser and can't hold configuration-time secrets)
  - confidential client applications (they run on servers and can hold configuration-time secrets)

## Code samples

### Application instantiation

- using `PublicClientApplicationBuilder`:

```csharp
IPublicClientApplication app = PublicClientApplicationBuilder.Create(clientId).Build();
```

- using `ConfidentialClientApplicationBuilder`:

```csharp
string redirectUri = "https://myapp.azurewebsites.net";
IConfidentialClientApplication app = ConfidentialClientApplicationBuilder.Create(clientId)
    .WithClientSecret(clientSecret)
    .WithRedirectUri(redirectUri )
    .Build();
```

### Builder modifiers

- `.WithAuthority` sets the application default authority to an Azure Active Directory authority

```csharp
var clientApp = PublicClientApplicationBuilder.Create(client_id)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenant_id)
    .Build();
```

- `.WithRedirectUri` overrides the default redirect URI

```csharp
var clientApp = PublicClientApplicationBuilder.Create(client_id)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenant_id)
    .WithRedirectUri("http://localhost")
    .Build();
```

- `.WithTenantId(string tenantId)`
- `.WithClientId(string)`
- `.WithComponent(string)`
- `.WithDebugLoggingCallback()`
- `.WithLogging()`
- `.WithTelemetry(TelemetryCallback telemetryCallback)`
- `.WithCertificate(X509Certificate2 certificate)`
- `.WithClientSecret(string clientSecret)`

### Example application

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.Identity.Client;

namespace az204_auth
{
    class Program
    {
        private const string _clientId = "APPLICATION_CLIENT_ID";
        private const string _tenantId = "DIRECTORY_TENANT_ID";

        public static async Task Main(string[] args)
        {
            var app = PublicClientApplicationBuilder
                .Create(_clientId)
                .WithAuthority(AzureCloudInstance.AzurePublic, _tenantId)
                .WithRedirectUri("http://localhost")
                .Build();
            string[] scopes = { "user.read" };
            AuthenticationResult result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();

            Console.WriteLine($"Token:\t{result.AccessToken}");
        }
    }
}
```

## General characteristics of Shared Access Signatures

- a SAS is a signed URI that points to one or more storage resources and includes a token that contains a special set of query parameters
- the token indicates how the resources may be accessed by the client
- one of the query parameters, the signature, is used to authorize access to the storage resource

## SAS Example

`https://medicalrecords.blob.core.windows.net/patient-images/patient-116139-nq8z7f.jpg?sp=r&st=2020-01-20T11:42:32Z&se=2020-01-20T19:42:32Z&spr=https&sv=2019-02-02&sr=b&sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D`

- components:
  - URI: `https://medicalrecords.blob.core.windows.net/patient-images/patient-116139-nq8z7f.jpg?`
  - SAS token: `sp=r&st=2020-01-20T11:42:32Z&se=2020-01-20T19:42:32Z&spr=https&sv=2019-02-02&sr=b&sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D`
    - `sp=r`: access rights
    - `st=2020-01-20T11:42:32Z`: start time
    - `se=2020-01-20T19:42:32Z`: expiry time
    - `sv=2019-02-02`: API version
    - `sr=b`: storage type
    - `sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D`: cryptographic signature

## Types of SAS

- user delegation SAS (recommended): secured with AAD credentials, delegates access to Blob storage
- service SAS: secured with the storage account key, delegates access to Blob storage, Queue storage, Table storage, or Azure Files
- account SAS: secured with the storage account key, delegates access to resources in one or more of the storage services

## Use cases

- when you want to provide secure access to resources in your storage account to any client who does not otherwise have permissions
- "presigned URL" workflow
- when copying a blob to another blob that resides in a different storage account
- when copying a file to another file that resides in a different storage account
- when copying a blob to a file or a file to a blob (always)

## Stored access policies

- a stored access policy lets you configure and manage shared access signatures
- to create or modify a shared access policy, call the `Set ACL` operation for the resource with a request body that specifies the terms of the access policy
  - creation may take up to 30 seconds
- to revoke a shared access policy you can delete it, rename it by changing the signed identifier, or change the expiry time to a value in the past
  - any form of cancellation takes effect immediately
- to remove all access policies from a resource, call the `Set ACL` operation with an empty request body
- C# .NET example:

```csharp
BlobSignedIdentifier identifier = new BlobSignedIdentifier
{
    Id = "stored access policy identifier",
    AccessPolicy = new BlobAccessPolicy
    {
        ExpiresOn = DateTimeOffset.UtcNow.AddHours(1),
        Permissions = "rw"
    }
};

blobContainer.SetAccessPolicy(permissions: new BlobSignedIdentifier[] { identifier });
```

- Azure CLI example:

```bash
az storage container policy create \
  --name <stored access policy identifier> \
  --container-name <container name> \
  --start <start time UTC datetime> \
  --expiry <expiry time UTC datetime> \
  --permissions <(a)dd, (c)reate, (d)elete, (l)ist, (r)ead, or (w)rite> \
  --account-key <storage account key> \
  --account-name <storage account name>
```

## General characteristics of Microsoft Graph

- it is the gateway to data and intelligence in Microsoft 365, Windows 10, and Enterprise Mobility + Security ![diagram](./../media/20230207154832.png)
- components:
  - the `https://graph.microsoft.com` API
  - Microsoft Graph **connectors** deliver data external to the Microsoft cloud into Microsoft Graph
  - Microsoft Graph **Data Connect** delivers Microsoft Graph data to Azure data stores

## Query Microsoft Graph by using REST

- unless explicitly specified in the corresponding topic, assume types, methods, and enumerations are part of the `microsoft.graph` namespace
- request structure: `{HTTP method} https://graph.microsoft.com/{version}/{resource}?{query-parameters}`
  - `PATCH` is used to update a resource with new values, while `PUT` replaces that resource entirely
  - the two currently supported versions are `v1.0` and `beta`
  - a resource can be an entity or complex type and can have relationships (examples: `me/messages`, `me/drive`, `me/sendMail`)
  - query parameters can be OData system query options or other strings that a method accepts to customize its response (example: `?filter=emailAddress eq 'jon@contoso.com'`)
- paginated responses provide the next page URL in `@odata.nextLink`

## Query Microsoft Graph by using SDKs

- the SDKs include two components: a service library and a core library
  - the service library contains models and request builders
  - the core library provides a set of features that enhance working with all the Microsoft Graph services (retry handling, secure redirects, transparent authentication, payload compression, paging through collections, creating batch requests)
- NuGet packages for .NET:
  - Microsoft.Graph: for accessing the `v1.0` API
  - Microsoft.Graph.Beta: for accessing the `beta` version of the API
  - Microsoft.Graph.Core: the core library

### Code examples

- creating a client

```csharp
var scopes = new[] { "User.Read" };

// Multi-tenant apps can use "common",
// single-tenant apps must use the tenant ID from the Azure portal
var tenantId = "common";

// Value from app registration
var clientId = "YOUR_CLIENT_ID";

// using Azure.Identity;
var options = new TokenCredentialOptions
{
    AuthorityHost = AzureAuthorityHosts.AzurePublicCloud
};

// Callback function that receives the user prompt
// Prompt contains the generated device code that you must
// enter during the auth process in the browser
Func<DeviceCodeInfo, CancellationToken, Task> callback = (code, cancellation) => {
    Console.WriteLine(code.Message);
    return Task.FromResult(0);
};

// https://learn.microsoft.com/dotnet/api/azure.identity.devicecodecredential
var deviceCodeCredential = new DeviceCodeCredential(
    callback, tenantId, clientId, options);

var graphClient = new GraphServiceClient(deviceCodeCredential, scopes);
```

- reading information

```csharp
// GET https://graph.microsoft.com/v1.0/me

var user = await graphClient.Me
    .Request()
    .GetAsync();
```

- retrieving a list of entities

```csharp
// GET https://graph.microsoft.com/v1.0/me/messages?$select=subject,sender&$filter=<some condition>&orderBy=receivedDateTime

var messages = await graphClient.Me.Messages
    .Request()
    .Select(m => new {
        m.Subject,
        m.Sender
    })
    .Filter("<filter condition>")
    .OrderBy("receivedDateTime")
    .GetAsync();
```

- deleting an entity

```csharp
// DELETE https://graph.microsoft.com/v1.0/me/messages/{message-id}

string messageId = "AQMkAGUy...";
var message = await graphClient.Me.Messages[messageId]
    .Request()
    .DeleteAsync();
```

- creating an entity

```csharp
// POST https://graph.microsoft.com/v1.0/me/calendars

var calendar = new Calendar
{
    Name = "Volunteer"
};

var newCalendar = await graphClient.Me.Calendars
    .Request()
    .AddAsync(calendar);
```

## Best practices

- use MSAL to acquire the access token to Microsoft Graph
- use the least privilege
- use the correct permission type based on scenarios
- consider who will be consenting to your application
- consider multi-tenant applications
- properly handle pagination and evolvable enumerations
- cache or store data locally only if necessary and that specific use case is covered by your terms of use and privacy policy, and does not violate the Microsoft APIs Terms of Use
