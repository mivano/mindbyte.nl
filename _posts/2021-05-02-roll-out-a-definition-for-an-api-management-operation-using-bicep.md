---
published: true
featured: false
comments: false
title: Roll out a definition for an API management operation using Bicep
tags:
  - IaC
  - Azure
  - API
  - Bicep
categories:
  - HTTP-APIs
---
I needed to roll out individual operations via infrastructure as code principles for API management in Azure using [Bicep](https://github.com/Azure/bicep) in a recent project.

The API Management instance was already available, so I only needed an API and a couple of operations. The operation (in this case, a POST call) needed a definition, a schema defining how the payload looks. The [documentation of Microsoft](https://docs.microsoft.com/en-us/azure/templates/microsoft.apimanagement/2019-01-01/service/apis/schemas?tabs=bicep) was not very descriptive how to add this, so here you go:

First I need an API:

```terraform
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

```terraform
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
          { "amount": 100 }
          '''        
          typeName: 'Transaction'
          schemaId: 'transaction'
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
  dependsOn:[
    transactionSchema
  ]
}
```

This adds the operation to the API and specifies the responses as well as the inputs. I use the `typeName` to reference the type inside the schema (transaction contains both **Transaction** and **Error** types).

So lets define that as well in Bicep:

```terraform
resource transactionSchema 'Microsoft.ApiManagement/service/apis/schemas@2019-01-01' = {
  name: '${apimanagementApi.name}/transaction'
  properties: {
    contentType: 'application/vnd.oai.openapi.components+json'
    document: {
      components: {
        schemas: {
          Transaction: {
            type: 'object'
            required: [
              'amount'
            ]
            properties: {
              id: {
                type: 'string'
                description: 'Internal transaction id'
                readOnly: true
              }           
              amount: {
                description: 'The amount that will be transferred'
                type: 'number'
                format: 'currency'
              }
              currency: {
                type: 'string'
                description: 'The currency used in this transaction (ISO 4217 3-letter currency code)'
                externalDocs: {
                  description: 'ISO 4217 currency codes'
                  url: 'https://www.iso.org/iso-4217-currency-codes.html'
                }
              }
            }
          }          
          Error: {
            required: [
              'code'
              'message'
            ]
            type: 'object'
            properties: {
              code: {
                type: 'integer'
                format: 'int32'
              }
              message: {
                type: 'string'
              }
            }
          }
        }
      }
    }
  }   
}

```

I use the [OpenAPI specification](https://swagger.io/docs/specification/data-models/) and can set the actual definition using the Bicep object notation (remember that this is not JSON, so no commas, add new lines, etc.). I could not get this to work with the latest API version.

The above allows you to create operations and definitions using Bicep and keep everything under source code.
