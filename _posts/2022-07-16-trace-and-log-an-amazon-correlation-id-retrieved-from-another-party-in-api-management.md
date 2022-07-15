---
published: true
title: Trace and log an amazon correlation id retrieved from another party in API Management
tags:
  - Azure
  - APIM  
---

A correlation Id is a great way to track requests over multiple boundaries and aids in troubleshooting. When you get an identifier from another party, it can be beneficial to use this value as a starting point and bring it along through the request. 
For a recent project, we had a party using Amazon services to send us webhooks. They used the by Amazon generated correlation Id (stored in a header called `x-amzn-requestid`) to track what they sent to us. So it made sense to persist that information somehow. With a policy in Azure API Management, we can extract, store, trace, and return this value.

```xml
<policies>
    <inbound>
        <base />
      
        <!--Use consumer correlation id or generate new one-->
        <set-variable name="correlation-id" value="@(context.Request.Headers.GetValueOrDefault("x-amzn-requestid", Guid.NewGuid().ToString()))"/>
      
        <!--Set header for end-to-end correlation-->
        <set-header name="x-correlation-id" exists-action="override">
          <value>@((string)context.Variables["correlation-id"])</value>
        </set-header>

        <!--Trace the correlation id-->
        <trace source="Webhook APIM Policy" severity="information">
          <message>@(String.Format("{0} | {1}", context.Api.Name, context.Operation.Name))</message>
          <metadata name="correlation-id" value="@((string)context.Variables["correlation-id"])"/>
        </trace>
        
        </inbound>
        <backend>
        <base />
        </backend>
    <outbound>
        <base />
        <choose>
          <when condition="@(context.Response.StatusCode == 201)">
           
              <return-response>
                  <set-status code="202" reason="Notification queued for processing" />
               
                  <!--Set header for end-to-end correlation-->
                  <set-header name="x-correlation-id" exists-action="override">
                    <value>@((string)context.Variables["correlation-id"])</value>
                  </set-header>
                 
              </return-response>
          </when>
        </choose>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

I removed some not interesting parts to highlight the relevant lines. We first extract the value from the `x-amzn-requestid` header or generate a new one when it does not exist. We set our own header that will go to the backend. We output the correlation id to the log with the [Trace policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#Trace). In my case, I have Application Insights [configured](https://mindbyte.nl/2021/05/14/link-appinsights-to-api-management-using-bicep.html) to log this information.

In the outbound section, I return a 202 response with the correlation id set in the header. This allows the sender to see the same id back in the response.

In Application Insights, you will find the id in the `correlation-id` field and you can search on the field as well.

![](/images/tracecorrelation.png)