---
published: true
featured: false
comments: false
title: Roll out a definition for an API management operation using Bicep
---
In a recent project I needed to roll out individual operations via infrastructure as code principles for API management in Azure using [Bicep](https://github.com/Azure/bicep). The API Management instance was already available, so I only needed an API and a couple of operations. The operation (in this case a POST call) needed a definition, a schema defining how the payload looks. The [documentation of Microsoft](https://docs.microsoft.com/en-us/azure/templates/microsoft.apimanagement/2019-01-01/service/apis/schemas?tabs=bicep) was not very descriptive how to add this.

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

I can now add operations to it:

```json
var operationName = '${apimanagementApi.name}/posttransaction'

resource apimanagementapioperationposttransaction 'Microsoft.ApiManagement/service/apis/operations@2020-06-01-preview' = {
  name: operationName
  properties: {
    description: 'Post method for sending a payment transaction.'
    displayName: 'Post a payment transaction.'
    method: 'POST' 
    urlTemplate: '/transactions'
    request: {
      description: 'Transaction to be processed'     
      representations: [
        {
          contentType: 'application/json'
          sample: '''
          { "transaction_id": "x" }
          '''        
          typeName: 'transaction'
        }
      ]
    }
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

This adds the operation to the API and specifies the reponses as well as the inputs. I reference the `typeName` here.

So lets define that as well in Bicep:

```json
resource transactionSchema 'Microsoft.ApiManagement/service/apis/schemas@2020-06-01-preview' = {
  name: '${apimanagementApi.name}/transaction${versionShort}'
  properties: {
    contentType: 'application/vnd.oai.openapi.components+json'
    document: {     
      definitions: {
        'transaction': {
          'type': 'object'
          'properties': {
              'transaction_id': {
                  'type': 'string'
                  'required': true
              }
            }
          }
      }
    }
  }
}

```

I use the [OpenAPI specification](https://swagger.io/docs/specification/data-models/) and can set the actual definition using the Bicep object notation (remember that this is not JSON, so no commas, add new lines etc). 

You will need to place the schema definition above the operation code or use a `dependsOn` to make sure it is created in the correct order.

This allows you to create operations and definitions using Bicep and keep everything under source code.


