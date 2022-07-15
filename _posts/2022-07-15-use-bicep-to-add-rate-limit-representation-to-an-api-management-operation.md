---
published: true
title: Use Bicep to add rate limit representation to an API Management operation
tags:
  - Azure
  - APIM  
  - Bicep
  - REST
---

In a [previous blog post](https://mindbyte.nl/http-apis/2021/05/02/roll-out-a-definition-for-an-api-management-operation-using-bicep.html) I showed how to roll out a definition for an API management operation using Bicep. Recently I added functionality for [rate limiting](https://mindbyte.nl/2022/06/18/use-different-rate-limiting-options-in-azure-api-management-for-business-and-non-business-hours.html) so it makes sense to add that to the API definition as well.

By default, Azure API Management returns a 429 (Too Many Requests) response when the rate limit is exceeded. It also includes a header with the number of seconds the client should wait before retrying. We can include this in the definition by adding a 429 status code response as shown below.

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
      {
        statusCode: 429
        description: 'Too many requests.'
        representations:[
          {
            contentType: 'application/json'
            sample: '''
            {
              "statusCode": 429,
              "message": "Rate limit is exceeded. Try again in 55 seconds."
            }
            '''
            typeName: 'RateLimit'
            schemaId: 'transaction'
          }
        ]
        headers: [
          {
            name: 'Retry-After'
            description: 'Time in seconds to wait before retrying the request.'
            type: 'integer'
          }
        ]
      }
    ]
  }
  dependsOn:[
    transactionSchema
  ]
}
```

We also need to extend the schema to include a RateLimit type.

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
          RateLimit: {
            type: 'object'
            properties: {
              statusCode: {
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

In the Developer Portal we can see that the schema is now available on the operation.

![](/images/ratelimitapi.png)