---
title: "Deploying AVD and retrieving the Registration Token with Bicep"
date: 2025-06-16 12:00:00 -500
categories: [Azure, AVD. Bicep]
tags: [AVd, Bicep, Azure Virtual Desktop, Registration Token, Infrastructure as Code]
author: CLC
description: Learn how to deploy AVD resources using Azure Bicep and retrieve the Host Pool registration token during the deployment to simplify the process.
toc: true
comments: true
media_subpath: /post-content/06-avd-biceptoken
---

## Overview

Azure Virtual Desktop (AVD) is a robust solution for delivering virtualized applications and desktops to users. To simplify the deployment and management of AVD environments, administrators often rely on Infrastructure as Code (IaC) tools like Azure Bicep.

One of the more nuanced challenges when deploying AVD with Bicep is handling the registration token required by session hosts. This token is typically short-lived, cannot be retrieved through a GET operation due to recent API changes, and must be passed securely into session host configuration steps such as DSC extensions.

Previously, certain API versions returned the registration token for a host pool on a GET (read) operation. While convenient, this raised a security concern: any user with Reader access could view an active token. To address this, Microsoft updated the AVD API in version `2023-09-05` so the registration token is only returned during a PUT or POST (write/update) operation. This change prevents passive access to registration tokens and aligns with the principle of least privilege. See [Microsoft's announcement](https://techcommunity.microsoft.com/discussions/azurevirtualdesktopforum/update-to-microsoft-desktop-virtualization-api-v-2023-09-05-by-august-2-2024-to-/4180665) for more.

Before the API change, it was easy to retrieve the registration token for a host pool by simply querying the host pool resource, and many deployment tools relied on this functionality. Now with the API updated, how can we still easily deploy a new Host Pool resource and retrieve the registration token?

In this post, we will discuss the use of two Bicep templates: `main.bicep` for deploying the AVD resources and `token.bicep` for generating and retrieving the registration token securely.

## Key Features of the Templates

You can find the full Bicep templates, parameter files, and example usage in the public GitHub repository:  
👉 [Az-AVD-Hostpool-Bicep](https://github.com/cocallaw/Az-AVD-Hostpool-Bicep)

### `main.bicep`

The `main.bicep` template provisions the core resources required for an AVD environment, including:

- **Host Pools**: Manage user sessions efficiently.
- **Session Hosts**: Virtual machines that host user sessions.
- **Application Groups**: Logical grouping of applications for users.
- **Workspaces**: Centralized access points for published resources.

### `token.bicep`

The `token.bicep` template securely generates and manages registration tokens. These tokens are essential for registering session hosts with the host pool.

## Why Use a Separate Token Template

Recent changes in the AVD API have enhanced security by preventing registration tokens from being retrieved using standard read/get operations. This ensures that tokens are only available during their creation, reducing the risk of unauthorized access.

By isolating token generation into a dedicated module (`token.bicep`), we can trigger the host pool update operation needed to create a new token without redeploying the entire environment. This separation also ensures better reusability and clarity in deployment flows.

### How the Token is Returned

The `token.bicep` file takes in the necessary information about the Host Pool resources that was created earlier in the template through the use of parameters and has the Host Pool generate an updated token by using the `registrationTokenOperation: 'Update'` property. The registration token is retrieved from the response by using the `listRegistrationTokens()` function on the `hostPoolTokenUpdate` resource.

#### main.bicep hostPoolRegistrationToken Module

```bicep
// Update the Host Pool to return Registration Token
module hostPoolRegistrationToken 'token.bicep' = {
  name: 'hostPoolRegistrationToken'
  params: {
    hostPoolName: hostPoolName
    tags: hostPool.tags
    location: hostPool.location
    hostPoolType: hostPool.properties.hostPoolType
    friendlyName: hostPool.properties.friendlyName
    loadBalancerType: hostPool.properties.loadBalancerType
    preferredAppGroupType: hostPool.properties.preferredAppGroupType
    maxSessionLimit: hostPool.properties.maxSessionLimit
    startVMOnConnect: hostPool.properties.startVMOnConnect
    validationEnvironment: hostPool.properties.validationEnvironment
    agentUpdate: hostPool.properties.agentUpdate
  }
  dependsOn: [
    desktopAppGroup
    workspace
  ]
}
```

#### token.bicep hostPoolTokenUpdate resource

```bicep
resource hostPoolTokenUpdate 'Microsoft.DesktopVirtualization/hostPools@2024-04-03' = {
  name: hostPoolName
  location: location
  tags: tags
  properties: {
    friendlyName: friendlyName
    hostPoolType: hostPoolType
    loadBalancerType: loadBalancerType
    preferredAppGroupType: preferredAppGroupType
    maxSessionLimit: maxSessionLimit
    startVMOnConnect: startVMOnConnect
    validationEnvironment: validationEnvironment
    agentUpdate: agentUpdate
    // Update the registration info with a new token
    registrationInfo: {
      expirationTime: dateTimeAdd(baseTime, tokenValidityLength)
      registrationTokenOperation: 'Update'
    }
  }
}
```

The `token.bicep` template outputs the registration token that is valid for the configured duration, defaulting to eight hours. This token must be used immediately after creation, as it cannot be retrieved later.

```bicep
@secure()
output registrationToken string = first(hostPoolTokenUpdate.listRegistrationTokens().value)!.token

```

### Why This Approach is Necessary

Restricting token retrieval aligns with best practices for secure token management. By ensuring that tokens are generated and consumed securely, this approach minimizes the risk of unauthorized access and enhances the overall security of the AVD environment.

To enhance the security of the deployment, the registration token can be stored as a secret in Azure Key Vault or another secure storage solution. This allows for controlled access to the token by different solutions or scripts that need to register session hosts with the host pool without exposing it in plain text.

## Deployment Instructions

### 1. Get the Bicep Templates

You can find the full Bicep templates, parameter files, and example usage in the public GitHub repository:  [Az-AVD-Hostpool-Bicep](https://github.com/cocallaw/Az-AVD-Hostpool-Bicep)

### 2 Install the Azure CLI and Bicep CLI

Ensure you have the Azure CLI and Bicep CLI installed on your machine. You can install them using the following commands:
[How to install the Azure CLI (Microsoft Learn)](https://learn.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest)
[How to install the Bicep CLI (Miceosoft Learn)](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install)

Once you have the Azure CLI and Bicep installed, sign in to your Azure account and set the appropriate subscription context:

```bash
az login
az account set --subscription <your-subscription-id>
```

### 2. Prepare Parameter File

Update the `main.bicepparam` file to provide deployment parameters specific to your environment.
```bicep
using 'main.bicep'

/*AVD Config*/
param hostPoolName = 'myAVDHostPool'
param sessionHostCount =  2
param maxSessionLimit =  5

/*VM Config*/
param vmNamePrefix =  'myAVDVM'
param adminUsername =  'superAdmin'
param adminPassword =  'NotaPassword!'

/*Network Config*/
param subnetName =  'mySubnet'
param vnetName =  'myVNet'
param vnetResourceGroup =  'myNetworkRG'
```

### 3. Deploy the Templates

Run the following command to deploy the entire solution, including token retrieval:

```bash
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file main.bicep \
  --parameters main.bicepparam
```

The `main.bicep` file internally calls the `token.bicep` module and injects the token into the session host configuration as part of the DSC extension setup.

## Conclusion

These Bicep templates simplify and secure the deployment of Azure Virtual Desktop environments. By retrieving the token as part of the deployment you can simplify your deployment process and in some cases reduce the number of steps needed.
