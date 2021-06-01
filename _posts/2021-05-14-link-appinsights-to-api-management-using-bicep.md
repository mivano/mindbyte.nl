---
published: true
featured: false
comments: false
title: Link Application Insights to Azure API Management using Bicep
tags:
  - IaC
  - Azure
  - API
  - Bicep
---
As it is good practice to monitor your system, I also wanted this to include the API Management side. It is allowing me to see the failed request in more detail and build my metrics. As we use infrastructure as code, I will use [Bicep](https://github.com/Azure/bicep) to configure this.

The API Management instance was already available, so I only needed an API and a couple of operations. The API management setup is shared between the teams, but I needed to use my own monitoring for my own endpoints. Let's also assume that we have an already rolled out Application Insights instance where we want to log the data.

First I need an API:

```json
resource apimanagementApi 'Microsoft.ApiManagement/service/apis@2020-06-01-preview' = {
  name: '${existingApiManagementName}/${apiName}'
  properties: {
    description: 'API definition.'
    type: 'http'   
    subscriptionRequired: true
    isCurrent: true
    displayName: 'My API'
    serviceUrl: 'https://x.y.windows.net/'
    path: 'api'
    protocols: [
      'https'
    ]
  }
}
```

The `existingApiManagementName` and `apiName` are variables used to reference the API management instance and give it a unique name.

Add an operation to it, so we have something to test with:

```json
var operationName = '${apimanagementApi.name}/posttransaction'

resource apimanagementapioperationposttransaction 'Microsoft.ApiManagement/service/apis/operations@2020-06-01-preview' = {
  name: operationName
  properties: {
    description: 'Post method for sending a payment transaction.'
    displayName: 'Post a payment transaction.'
    method: 'POST' 
    urlTemplate: '/transactions'
    responses: [
      {
        statusCode: 202
        description: 'Transaction is accepted for validation and processing.'
      }
      {
        statusCode: 403
        description: 'Unauthorized call.'
      }
    ]
  }
}
```

In API Management you will need to register your Application Insights instance before using it to send data to. I need the Id of the instance as well as the instrumentation key. You can either get them from the properties if you roll out Application Insights as well, or fetch them using a reference. In my case, I passed them in via a parameter. To fetch them with the az cli:

```bash
appinsightsId=$(az resource show --name nameofyourinstance -g yourresourcegroup --resource-type 'microsoft.insights/components' -o tsv --query id)
appinsightsKey=$(az resource show --name nameofyourinstance -g yourresourcegroup --resource-type 'microsoft.insights/components' -o tsv --query properties.InstrumentationKey)
       
```

And now we register the logger with as name `your-applicationinsights`. The instrumentationKey is placed as a **Named values** secret value automatically.

```json
resource appInsightsAPIManagement 'Microsoft.ApiManagement/service/loggers@2020-06-01-preview' = {
  name: '${existingApiManagementName}/your-applicationinsights'
  properties: {
    loggerType: 'applicationInsights'
    description: 'Your specific Application Insights instance.'
    resourceId: appInsightsId
    credentials: {
      instrumentationKey: appInsightsKey
    }
  }
}
```

We can now link the API with the logger:

```json
resource apiMonitoring 'Microsoft.ApiManagement/service/apis/diagnostics@2020-06-01-preview' = {
  name: '${apimanagementApi.name}/applicationinsights'
  properties: {
    alwaysLog: 'allErrors'
    loggerId: appInsightsAPIManagement.id  
    logClientIp: true
    httpCorrelationProtocol: 'W3C'
    verbosity: 'information'
    operationNameFormat: 'Url'
  }
}
```

The `loggerId` is a reference to the configured logger. It is also essential to use the name `/applicationinsights` as you configure this logger explicitly as this type (another option is the `azuremonitor` type).

I included some basic settings, but you can find more options in the [documentation](https://docs.microsoft.com/en-us/azure/templates/microsoft.apimanagement/2019-01-01/service/apis/diagnostics?tabs=bicep).
When you now start hitting your API POST endpoint, you will get traces inside Application Management within a few minutes.

## Updated 1 June 2021

Although the above method works fine, it does create a new named value each time you run the deployment. So you can end up with a lot of records called `Logger-Credentials-*` which contain the key. In order to avoid this, create your own named value and reference the value instead. It will not create a new named value.

```json
// We store the appinsights key to a named value and then make a reference to it
resource namedValueAppInsightsKey 'Microsoft.ApiManagement/service/namedValues@2020-06-01-preview' = {
  name: '${existingApiManagementName}/appinsights-key'
  properties: {
    tags: []
    secret: false
    displayName: 'appinsights-key'
    value: appInsightsKey
  }
}

// Add Application Insights to API management
resource appInsightsAPIManagement 'Microsoft.ApiManagement/service/loggers@2020-06-01-preview' = {
  name: '${existingApiManagementName}/${environment}-applicationinsights'
  properties: {
    loggerType: 'applicationInsights'
    description: 'app specific Application Insights instance.'
    resourceId: appInsightsId
    credentials: {
      instrumentationKey: '\{{appinsights-key}\}'
    }
  }
  dependsOn:  [
    namedValueAppInsightsKey
  ]
}
```

If you do have a large number of values you want to remove, you can use the below PowerShell script.

```powershell
$sub = 'fill in subscription id'
$name = 'fill in name of api management instance'
$rg = 'fill in name of resource group'

$v = az apim nv list -n $name -g $rg --subscription $sub -o json | ConvertFrom-Json 

$v | foreach { 
    if ($_.displayName  -like 'Logger-Credentials-*') {
        Write-Output $_.displayName
        az apim nv delete -n $name -g $rg --subscription $sub --named-value-id $_.name -y
    }
}
```
