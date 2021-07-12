---
published: true
featured: false
title: Provision an Azure Function plan with a subnet using Bicep
tags:
  - IaC
  - Azure
  - Bicep
  - GitHub
---

In a recent project, I need an Azure Function hosted inside a subnet. To have vnet support, I need to use a premium plan, and I wanted to use Linux hosting as well. All this must be part of my CICD rollout using [Bicep](https://github.com/Azure/bicep).

First of all, I need to roll out a hosting plan placed inside a Bicep module. A module is nothing more than a separate file containing the resource definition, some parameters, and output.

```terraform
@description('Name of hosting plan')
param hostingPlanName string

param location string = resourceGroup().location

@allowed([
  'Y1'
  'EP1'
  'EP2'
  'EP3'  
])
@description('The name of the SKU to use when creating the Azure Functions plan. Common SKUs include Y1 (consumption) and EP1, EP2, and EP3 (premium).')
param functionPlanSkuName string = 'EP1'

var functionPlanKind = functionPlanSkuName ==  'Y1' ?  'functionapp' : 'elastic'

resource hostingPlan 'Microsoft.Web/serverfarms@2020-06-01' = {
  name: hostingPlanName
  location: location
  properties: {    
    reserved: true    
  }
  kind: functionPlanKind
  sku: {
    name: functionPlanSkuName    
  }
}

output id string = hostingPlan.id
```

The module will create a place to host the application, so let's create another module that will roll out a function app.

```terraform
@description('Id of hosting plan')
param hostingPlanId string

param functionAppName string

param location string = resourceGroup().location

@description('The application runtime that the function app uses')
param functionRuntime string = 'dotnet'

param functionSubnetId string

param environmentName string

param instrumentationKey string

param storageAccountName string

var connectionString = concat('DefaultEndpointsProtocol=https;AccountName=${storageAccountName};AccountKey=', listKeys(resourceId('Microsoft.Storage/storageAccounts', storageAccountName), '2019-06-01').keys[0].value)

resource functionApp 'Microsoft.Web/sites@2020-12-01' = {
  name: functionAppName
  kind: 'functionapp'
  location: location  
  properties: {
    enabled: true    
    serverFarmId: hostingPlanId
    reserved: true    
    siteConfig: {
      detailedErrorLoggingEnabled: false
      vnetRouteAllEnabled: true

      ftpsState: 'Disabled'
      http20Enabled: true

      minTlsVersion: '1.2'
      scmMinTlsVersion: '1.2'
      minimumElasticInstanceCount: 1
    } 
    hostNamesDisabled: true
    httpsOnly: true    
  }
  identity: {
    type: 'SystemAssigned'    
  }  
}

// Add the function to the subnet
resource functionToSubnet 'Microsoft.Web/sites/networkConfig@2016-08-01' = {
  name: '${functionAppName}/VirtualNetwork'
  properties: {
    subnetResourceid: functionSubnetId
    swiftSupported: true
  }
  dependsOn:[
    functionApp
  ]
}

resource functionAppSettings 'Microsoft.Web/sites/config@2020-12-01' = {
    name: '${functionAppName}/appsettings'
    properties: {
        'FUNCTIONS_EXTENSION_VERSION':  '~3'
        'FUNCTIONS_WORKER_RUNTIME': functionRuntime
        'ASPNETCORE_ENVIRONMENT': environmentName
        'APPINSIGHTS_INSTRUMENTATIONKEY': instrumentationKey
        'WEBSITE_VNET_ROUTE_ALL': '1'
        'WEBSITE_CONTENTOVERVNET': '1'
        'AzureWebJobsDisableHomepage': 'true'
        'AzureWebJobsStorage': connectionString
    }
    dependsOn:[
        functionApp
        functionToSubnet
    ]
}

output principalId string = functionApp.identity.principalId
```

It will create a function app inside the hosting plan, enable it for a `functionapp`, set a couple of options, and create a system-assigned identity.

The next part is adding the function to the subnet specified in the `functionSubnetId` parameter. You will not find this in the [Microsoft documentation](https://docs.microsoft.com/en-us/azure/templates/microsoft.web/sites?tabs=bicep), and it is tempting to use `Microsoft.Web sites/virtualNetworkConnections` instead. However, this creates a different kind of connection, not linking the function to the subnet.

The last part sets a couple of application settings. You can also inline this when creating the function itself. I specify with `WEBSITE_VNET_ROUTE_ALL` and `WEBSITE_CONTENTOVERVNET` that all network traffic needs to go over the virtual network. 

Normally, I can also specify a `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING` and `WEBSITE_CONTENTSHARE`, but that will only work when they reference a storage account that is not behind a virtual network. According to [GitHub](https://github.com/Azure/Azure-Functions/issues/1361#issuecomment-571700656) I can omit these settings and get some internal storage. 

Now we combine this by calling the above modules:

```terraform
// Create hosting plan
module hostingplan './modules/hostingplan.bicep' = {
  name: 'hostingplan'
  scope: resourceGroup
  params: {
    functionPlanSkuName: 'EP1'
    hostingPlanName: '${environment}-functionplan'   
  }
}

// Create functions
module monitoringFunctions './modules/function.bicep' = {
  name: 'monitoring'
  scope: resourceGroup
  params: {    
    hostingPlanId: hostingplan.outputs.id
    functionAppName: '${environment}-monitoring'
    functionSubnetId: '${virtualNetworkId}/subnets/functions-${environment}' 
    environmentName: environment
    instrumentationKey: applicationInsights.outputs.instrumentationKey
    storageAccountName: 'storageaccountname'
  }
  dependsOn:[    
    hostingplan
  ]
}

``` 

The function will need a storage account for its internals like durable functions, so you need to either create or reference it. Same with the instrumentation key of Application Insights.
I have a shared network between different environments, so that is why I postfix it with the environment name. The id is passed in via a parameter.

Now we only add a single function to the plan, but I can reuse it to add more.

To deploy your function app, you can use something like the below GitHub actions pipeline code:

```yaml
name: Deploy code 

env:
  AZURE_FUNCTIONAPP_PACKAGE_PATH: 'src/code'    # set this to the path to your web app project
  DOTNET_VERSION: '3.1.x'              # set this to the dotnet version to use
  OUTPUT_PATH: ${{ github.workspace }}/.output

# Controls when the action will run. 
on:
  push:
    branches: 
    - main
  pull_request:
    branches: 
    - main
    - 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@main

    - name: Setup DotNet ${{ env.DOTNET_VERSION }} Environment
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: 'Resolve Project Dependencies Using Dotnet'
      shell: bash
      run: |
        pushd './${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}'
        dotnet publish --configuration Release --output ${{ env.OUTPUT_PATH }}
        popd
   
    - name: Package functions
      uses: actions/upload-artifact@v1
      with:
          name: functions
          path: ${{ env.OUTPUT_PATH }}

  deploy_test:
    if: github.event_name == 'pull_request'
    name: Deploy to Test 
    runs-on: ubuntu-latest
    needs: [build]
    env:
        FUNC_APP_NAME_MONITORING: tst-monitoring
    steps:
      - name: Download functions
        uses: actions/download-artifact@v1
        with:
            name: functions
            path: ${{ env.OUTPUT_PATH }}

      - name: "Login via Azure CLI"
        uses: azure/login@v1
        with:
            creds: ${{ secrets.OTA_CONTRIBUTOR_CREDENTIALS }}

      - name: "Run Azure Functions Action for monitoring"
        uses: Azure/functions-action@v1
        with:
            slot-name: 'production'
            app-name: ${{ env.FUNC_APP_NAME_MONITORING }}
            package: ${{ env.OUTPUT_PATH }}

      - name: "Set Function app config" 
        uses: azure/appservice-settings@v1
        with:
          app-name: ${{ env.FUNC_APP_NAME_MONITORING }}
          mask-inputs: true          
          app-settings-json: '[
          { 
            "name": "TaskHubName",
            "value": "monitoring20210525", 
            "slotSetting": false
          },
          { 
            "name": "Blob__RelationsContainer",
            "value": "salesforce",
            "slotSetting": false
          }
              ]'
        id: settings
   
```

This yaml contains a build and a deploy stage and will deploy a published function from the build stage and apply settings.
