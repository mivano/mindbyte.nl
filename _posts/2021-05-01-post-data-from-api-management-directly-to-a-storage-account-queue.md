---
published: true
featured: false
title: POST data from API Management directly to a storage account queue
tags:
  - Azure
  - API
  - Bicep
categories:
  - HTTP-APIs
---
When you want to accept data, queue it up and process it at a later stage, you might want to use a simple way to push data to the queue without building a backend service.

With API Management, this is possible; have a POST operation that delivers the payload directly to the queue using a policy. For this, we use the [PUT Message](https://docs.microsoft.com/en-us/rest/api/storageservices/put-message) API call on the storage account.

First of all, you will need a storage account and an API management instance in Azure. In API Management, create a new API (the web service url does not matter) and a new operation inside this API. The operation will be using a POST method and use a sensible URL.

In the newly created operation, you can now edit the policy. Copy and paste the below:

```xml
<policies>
    <inbound>
        <base />
      
        <set-variable name="UTCNow" value="@(DateTime.Now.ToString("R"))" />
        <set-header name="Authorization" exists-action="override">
            <value>@{

            string accountName = "name-of-your-storage-account";
            string sharedKey = "access-key-of-storage-account";
            string sig = "";
            string contentType = context.Request.Headers.GetValueOrDefault("Content-Type");

            string canonical = String.Format("POST\n\n{0}\n\nx-ms-date:{1}\n/{2}/incoming-data/messages", contentType, context.Variables.GetValueOrDefault<string>("UTCNow"), accountName );

            using (HMACSHA256 hmacSha256 = new HMACSHA256( Convert.FromBase64String(sharedKey) ))
            {
                Byte[] dataToHmac = Encoding.UTF8.GetBytes(canonical);
                sig = Convert.ToBase64String(hmacSha256.ComputeHash(dataToHmac));
            }

            return String.Format("SharedKey {0}:{1}", accountName, sig);

            }</value>
        </set-header>
        <set-header name="x-ms-date" exists-action="override">
            <value>@( context.Variables.GetValueOrDefault<string>("UTCNow") )</value>
        </set-header>
        <set-backend-service base-url="https://name-of-your-storage-account.queue.core.windows.net/" />
        <set-body>@{ 
            JObject inBody = context.Request.Body.As<JObject>(); 
            
            return  "<QueueMessage><MessageText>"+inBody.ToString()+"</MessageText></QueueMessage>"; 
        }</set-body>
        <rewrite-uri template="incoming-data/messages" copy-unmatched-params="true" />
        <!--  Don't expose APIM subscription key to the backend. -->
        <set-header name="ocp-apim-subscription-key" exists-action="delete" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <choose>
            <when condition="@(context.Response.StatusCode == 201)">
                <return-response>
                    <set-status code="202" reason="Transaction queued for processing" />
                </return-response>
            </when>
        </choose>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

A couple of things are happening here:

With the `set-backend-service` we set the new backend URL. In this case, the storage account base URL. We also need to send the body in a specific format, as you can see in the [Microsoft documentation](https://docs.microsoft.com/en-us/rest/api/storageservices/put-message#request-body). The `set-body` allows that transformation.

To specify which queue, we `rewrite-uri` and make sure to add the `/messages` endpoint to it.

Next is the authorization. The whole process is explained on the [docs site](https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-with-shared-key) and involves signing a combined string using the account key.

We know we do a POST call, we fetch the content-type header value, generate our own `x-ms-date` header (and put in variable as we need to output it in the request as well) and we end with the relative endpoint.
So that leads to this canonical form:

```csharp
string canonical = String.Format("POST\n\n{0}\n\nx-ms-date:{1}\n/{2}/incoming-data/messages", contentType, context.Variables.GetValueOrDefault<string>("UTCNow"), accountName );
```

The `Authorization` header will be set with the value of `return String.Format("SharedKey {0}:{1}", accountName, sig);`

The last part is in the `outbound`; the PUT operation on the storage account returns a 201 (created). However, we wanted to produce a 202 (accepted and queued for processing). So we change the status to 202 when we see a 201.

In the policy, you see a couple of variables. Of course you do not want those exposed inside the policy, so replace them by double curly braces ({ and }) and add them to the Named Values of API management. Make sure to use a secret (or Keyvault reference) for the accesskey variable.
