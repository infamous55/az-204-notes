---
title: Azure App Service
created_at: 02-01-2023
tags: [azure, az-204]
---

## General characteristics of Azure App Service

- HTTP-based service for hosting web applications, REST APIs, and mobile backends
- supports CI/CD
- offers an Authentication and Authorization service
- has various configuration options
- supports autoscaling, both up/down and in/out
- an App Service Plan can have several deployment slots

## Limitations of Linux resources in Azure App Service

- can't be used with the shared pricing tier
- can't be mixed with Windows plans
- can't use App Service Authentication and Authorization with them because they run on separate containers
- not all runtimes are available on Linux: `az webapp list-runtimes --os-type linux`

## Azure App Service Plans

- each plan defines:
  - region
  - number of VMs
  - the size of VMs
  - pricing tier
- a single plan can host multiple applications

### Pricing tiers

- the pricing tiers can fall into a few categories:
  - shared compute: Free and Shared pricing tiers, only suitable for development and testing, can't scale out
  - dedicated: Basic, Standard, Premium, PremiumV2, PremiumV3
  - Isolated: same as dedicated + network isolation
  - Consumption: only when ASP is used for Azure Functions
- the pricing tier of an ASP can be changed as long as the current setup is supported

## Deployment

- automated:
  - Azure DevOps
  - GitHub
  - Bitbucket
- manual:
  - Git
  - CLI: `az webapp up`
  - Zip/War files
  - FTP(S)

## App Service Authentication and Authorization

- not mandatory, but is available as an option
- can be integrated with various identity providers
  - default: Microsoft, Facebook, Google, Twitter
  - plus Any OpenID Connect provider
- runs in a separate container, no SDK needed
- there are two authentication flows:
  - server-directed flow:
    - app redirects the user to the sign-in endpoint `/.auth/login/<provider_name>`
    - the provider redirects to `/.auth/login/<provider_name>/callback`
    - the cookie is added and handled by the browser in the following requests
  - client-directed flow:
    - client signs the user in with the provider's SDK
    - client posts the token to `/.auth/login/<provider_name>` for validation
    - App Service returns its authentication token
    - client code presents an authentication token in the `X-ZUMO-AUTH` header
- can allow unauthenticated requests to reach the web app, or can require authentication (not suitable for single-page applications, and returns `HTTP 401 Unauthorized` by default)

## App Service Networking

- apps are accessible over the Internet by default
- App Service is a distributed system:
  - front ends handle the incoming HTTP(S) traffic
  - workers handle the customer workload
- the Free, Shared, Basic, Standard, and Premium plans all use the same worker VM type
- the PremiumV2, and PremiumV3 plans each use a different worker VM type
- when you change the VM type, you get a different set of outbound addresses
- `az webapp show \ --resource-group <group_name> \ --name <app_name> \ --query outboundIpAddresses \ --output tsv` gives all current outbound addresses
- `az webapp show \ --resource-group <group_name> \ --name <app_name> \ --query possibleOutboundIpAddresses \ --output tsv` gives all possible outbound addresses
- unless it is isolated, the App Service network can't be connected to an on-premises network, but there are inbound and outbound features that can be used
  - Inbound features:
    - App-assigned address
    - Access restrictions
    - Service endpoints
    - Private endpoints
  - Outbound features:
    - Hybrid Connections
    - Gateway-required virtual network integration
    - Virtual network integration

## Application Settings Configuration

- they are environment variables
- can be configured through the GUI or as JSON files
- nested key structures are written with a double underscore
- connection string also has a type property
- the general settings are:
  - Stack Settings (runtime, SDK version, a custom entry command or file for container apps)
  - Platform Settings (WebSockets Protocol, HTTP version, etc.)
  - Debugging (automatically turns off after 48 hours)
  - Incoming client certificates (require TLS mutual authentication)

### Handler mappings and storage extension

- for Windows apps:
  - you can customize IIS handler mappings to add custom script processors to handle requests for specific file extensions
    - to configure a handler you need to specify the extension, the script processor path, and optional command-line arguments
  - you can configure virtual applications and directories by specifying each virtual directory and its corresponding physical path relative to the website root (`D:\home\site\wwwroot`)
- you can add custom storage: Azure Blobs or Azure Files on Linux, and only Azure Files on Windows

### Diagnostic logging

- Application logging and Deployment logging are available on both Windows and Linux
- Web server logging, Detailed error logging, and Failed request tracing are only available on Windows
- there are different levels of logging available for Windows in the Azure Portal, under App Service logs
- you can specify the disk quota for the application logs if they run on File System and a value for the Retention Period
- you can stream logs within the Azure Portal, or by using the CLI: `az webapp log tail --name appname --resource-group myResourceGroup`
- to access logs stored in Azure Storage blobs use the standard tools
- to read logs stored in the App Service file system, the easiest way is to download the ZIP file in the browser at:
  - `https://<app-name>.scm.azurewebsites.net/api/logs/docker/zip` for Linux/container apps
  - `https://<app-name>.scm.azurewebsites.net/api/dump` for Windows apps

### Security certificates

- there are multiple options to add a certificate in App Service:
  - create a free App Service managed certificate
  - purchase an App Service certificate (offers renewal and export options)
  - import a certificate from Key Vault
  - upload a private certificate
  - upload a public certificate
- if you want to upload a certificate, it must meet some strict requirements
- you can't create a free certificate in the shared compute pricing tiers
- the free certificate has some limitations
- certificates are purchased from GoDaddy, kept in Azure Key Vault, and synchronized with App Service apps
- uploaded certificates need to be in the PFX format
- there is an option to enforce HTTPS when you do have a TLS certificate

### App features

- a feature flag is a variable with a binary state of on or off made available within the application code
- a feature manager is an application package that handles the lifecycle of all the feature flags in an application
- a filter is a condition to evaluate the state of a feature (user group, device, location, time window, percentage of traffic, etc.)
- the feature manager supports `appsettings.json` as a configuration source for feature flags
- Azure App Configuration is a centralized repository for feature flags

## Autoscale

- autoscale performs scaling in and out, only changing the number of servers running
- adds or removes servers based on a schedule or the following metrics:
  - CPU percentage
  - Memory percentage
  - Disk Queue Length (the total number of I/O requests)
  - HTTP Queue Length (how many client requests are waiting for processing)
  - Data In (number of bytes received)
  - Data Out (number of bytes sent)
- it isn't a good approach to handling long-term growth
- how an autoscale rule analyzes metrics:
  - it aggregates metrics for all instances across the time grain (usually 1 minute)
  - then calculates the Average/Minimum/Maximum/Sum/Last/Count of the time aggregation
  - after that, autoscale performs another aggregation of the values across the set Duration (minimum 5 minutes)
  - and yet another calculation that can differ from the one used for the time grain
- autoscale rules should be paired to handle both scaling-in and scaling-out as autoscale conditions can contain several rules
- on scale-out, autoscale runs if any rule is met, on scale-in, autoscale requires all rules to be met
- autoscale actions use a numeric comparison operator to determine how to react to the threshold being met
- autoscale actions have a cool-down period to stabilize the system between scaling operations
- autoscale actions can increase/decrease the current number of VMs or set it to a specific value
- shared pricing tiers don't support this type of scaling, while the Basic tier offers only one server instance, making autoscaling not possible
- best practices:
  - ensure to have appropriate values for the maximum, minimum, and default number of VMs
  - choose thresholds carefully, knowing that autoscale is designed to avoid flapping

## Deployment slots

- the Standard tier supports up to 5 slots, while the Premium and Isolated tiers support up to 20
- swapping changes the settings and the host names to redirect traffic which avoids causing any downtime
- after a swap, the slot with the previously staged app has the previous production app, making it easy to roll back
- there are slot-specific app settings that stick to a slot
- you can swap with preview, pausing the process after changing the settings to validate everything works correctly
- after preview (if it is the case), the source slot performs a complete restart
- all instances are warmed up by automatically making an HTTP request to the application root to trigger local cache initialization
- local cache initialization causes another restart on each instance
- you can also specify an URL for custom warm-ups
- a warm-up is considered complete if the app responds with a valid HTTP code
- some settings can't be swapped:
  - Publishing endpoints
  - Custom domain names
  - Non-public certificates and TLS/SSL settings
  - Scale settings
  - WebJobs schedulers
  - IP restrictions
  - Always On
  - Diagnostic log settings
  - Cross-origin resource sharing (CORS)
- `WEBSITE_OVERRIDE_PRESERVE_DEFAULT_STICKY_SLOT_SETTINGS` makes all settings swappable or not swappable
- if auto swap is enabled, every time you push your code changes to that slot, App Service automatically swaps the app into production after it's warmed up in the source slot
- auto swap isn't supported on Linux
- you can route a percentage of traffic to a non-production/staging slot, and that request will have the `x-ms-routing-name=staging` cookie, while the requests routed to the production slot will have the `x-ms-routing-name=self` cookie
- this specific type of routing can be done manually with a link that has the `?x-ms-routing-name=` query
