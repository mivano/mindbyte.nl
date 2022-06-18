---
published: true
title: Use different rate-limiting options in Azure API Management for business and non-business hours
tags:
  - Azure
  - API
  - APIM  
  - Bicep
---

Using API Management in front of your application has many benefits; one is using rate limiting. Rate limiting is a way to limit the number of requests that can be made to an API. Limiting the number of request is useful when for example, your backend service cannot handle a high amount of calls. When the caller sends more requests than allowed, the API returns a 429 (too many requests) response and indicates when the client can try again.

Although rate limiting is a very common scenario, and API Management has policies out of the box for this, I had a different requirement. We had a party that wanted to send historical data to our API. A large number of items which will take a long time to deliver all to us. We also had our regular traffic during the day to our API, which is more time-sensitive, so we did not want to wait on those requests while processing the historical load. We did want to use the same endpoints as the data, and the flow was the same. Loading the historical data was a typical thing for outside business hours.

I created a new product for this import purpose and linked it to the API. We have a policy on this product, so it will only be applied when the API is accessed with a subscription key. This policy uses a condition to select either the inside or outside business hours rate limit. So when the historical data importer is sending data during a normal business day, it will only be allowed a subset of the full load and the underlying API is saved from these request. When the client handles the 429 code correctly, it can automatically determine when to try again, so it does not need an understanding of the exact times.

```xml
<policies>
    <inbound>
        <set-variable name="inside" value="@{ return int.Parse("{{calls-per-minute-import-inside-business-hours}}"); }" />
        <set-variable name="outside" value="@{ return int.Parse("{{calls-per-minute-import-outside-business-hours}}"); }" />
        <choose>
            <when condition="@((DateTime.Now >= DateTime.Now.Date.AddHours(7) && DateTime.Now < DateTime.Now.Date.AddHours(18)) && !(DateTime.Now.ToString("dddd") == "Saturday" || DateTime.Now.ToString("dddd") == "Sunday") )">
                <rate-limit-by-key calls="@( (int)context.Variables["inside"])" renewal-period="60" counter-key="@(context.Subscription.Id)" />
            </when>
            <otherwise>
                <rate-limit-by-key calls="@( (int)context.Variables["outside"])" renewal-period="60" counter-key="@(context.Subscription.Id)" />
            </otherwise>
        </choose>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

In the policy, I define two variables to get the rate limits for the inside and outside business hours from named values. You can also directly place the numbers inside the policy, but this gives an excellent way to change the rate limits without having to change the policy. Adding it directly in the rate limit part did not work. It is sensitive about the type you put in there, so some casting is needed.

The condition test for a time between 7 and 18 hours and not on a weekend. It would be great if we could use the DayOfWeek function in the condition, but I could not get this to work as it returned an error about Enums not valid in an expression.

So play with the numbers and the date range, apply the policy with for example Bicep:

```terraform
resource apimImportProductPolicy 'Microsoft.ApiManagement/service/products/policies@2020-06-01-preview' = {
  name: '${apimanagementImportProduct.name}/policy'
  properties: {
    format: 'rawxml'
    value: '''
    <policies>
        <inbound>
            <set-variable name="inside" value="@{ return int.Parse("{{calls-per-minute-import-inside-business-hours}}"); }" />
            <set-variable name="outside" value="@{ return int.Parse("{{calls-per-minute-import-outside-business-hours}}"); }" />
            <choose>
                <when condition="@((DateTime.Now >= DateTime.Now.Date.AddHours(7) && DateTime.Now < DateTime.Now.Date.AddHours(18)) && !(DateTime.Now.ToString("dddd") == "Saturday" || DateTime.Now.ToString("dddd") == "Sunday") )">
                    <rate-limit-by-key calls="@( (int)context.Variables["inside"])" renewal-period="60" counter-key="@(context.Subscription.Id)" />
                </when>
                <otherwise>
                    <rate-limit-by-key calls="@( (int)context.Variables["outside"])" renewal-period="60" counter-key="@(context.Subscription.Id)" />
                </otherwise>
            </choose>
            <base />
        </inbound>
        <backend>
            <base />
        </backend>
        <outbound>
            <base />
        </outbound>
        <on-error>
            <base />
        </on-error>
    </policies>
          '''
  }  
  dependsOn:[
    namedValueInsideBusinessHours
    namedValueOutsideBusinessHours
  ]
}
```

And for the named values

```terraform
resource namedValueInsideBusinessHours 'Microsoft.ApiManagement/service/namedValues@2020-06-01-preview' = {
  name: '${existingApiManagementName}/calls-per-minute-import-inside-business-hours'
  properties: {
    tags: []
    secret: false
    displayName: 'calls-per-minute-import-inside-business-hours'
    value: '10'
  }
}

resource namedValueOutsideBusinessHours 'Microsoft.ApiManagement/service/namedValues@2020-06-01-preview' = {
  name: '${existingApiManagementName}/calls-per-minute-import-outside-business-hours'
  properties: {
    tags: []
    secret: false
    displayName: 'calls-per-minute-import-outside-business-hours'
    value: '500'
  }
}
```

