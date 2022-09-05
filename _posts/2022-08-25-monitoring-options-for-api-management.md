---
published: true
title: Monitoring options for API Management
tags:
  - API
  - APIM  
---
Monitoring your API Management operations is important. It gives you insight in the usage, the load and performance. But also helps you troubleshoot issues and correlate requests. So what are the options; 

1️⃣There is built-in analytics, keeping track of APIs and their operations, including locations, users and requests. There is some basic filtering and time range selection.

![https://pbs.twimg.com/media/Fa_r80gWIAATZk4.png](https://pbs.twimg.com/media/Fa_r80gWIAATZk4.png)

2️⃣Send the data to Application Insights, which can be selected even on API level. Requests, calls to back-end, exceptions and your own traces from policies are stored. Use emit-metric to include metrics. There is a performance cost, so use sampling. See [https://mindbyte.nl/2021/05/14/link-appinsights-to-api-management-using-bicep.html](https://mindbyte.nl/2021/05/14/link-appinsights-to-api-management-using-bicep.html)

3️⃣Log to Azure Event Hub, which can handle a large amount of data. Process this data via different tools to your needs or send it to third party tools. [https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-log-event-hubs](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-log-event-hubs)

4️⃣Store the data in Log Analytics workspace using Azure Monitor option. This exposes the metrics listed here: [https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-supported#microsoftapimanagementservice](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-supported#microsoftapimanagementservice)
and can be searched and alerted on.

As you can see, there are various options, from built-in to more advanced scenarios. 