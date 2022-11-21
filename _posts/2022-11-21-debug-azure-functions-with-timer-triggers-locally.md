---
published: true
title: Debug Azure Functions with timer triggers locally
tags:
  - Azure
  - Functions
  - Coding 
---

I recently saw a blog post on how to debug Azure Functions which run on something like a timer trigger. As you must wait for the timer to kick off, it is hard to step into your code at the right time. The proposed solution involved setting up a HttpTrigger, which would feed a queue trigger to reach the code eventually...complex, and a lot of additional code is needed. Let alone adding another component in the mix, the queue. There is however a better way...

Let us create a simple Timer trigger, as shown in the screenshot. This one runs every 5 minutes, so if you attach a debugger, you will have to wait 5 mins. But luckily, there is a solution!

![](/images/timertrigger.jpg)

You can use the Function Admin API to call the trigger directly. There is no need for explicit Http triggers; the admin API allows you to call any trigger by executing a POST with a payload. You can pass in input parameters or leave them empty like `{ }`. You do need to set the content-type to `application/json`

![](/images/functionapipost.jpg)

This trick will also work when your function is deployed, but you need the master key in the `x-functions-key` header. 

The [documentation](https://learn.microsoft.com/en-us/azure/azure-functions/functions-manually-run-non-http) page contains some more pointers, but it is relatively easy to call functions even with specific parameters and gives you control over your function triggers.
