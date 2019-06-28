---
published: false
featured: false
comments: false
title: How to monitor a 24 hour global event
description: How to monitor a 24 hour global event
tags:
  - azure
  - gdbc
---
For the last three years, Xpirit and Solidy organize the [Global DevOps Bootcamp](https://globaldevopsbootcamp.com/), an online event that teaches people about the DevOps practices using Microsoft technologies. As the name global  implies, the event takes place on the same day for around 100 venues all over the world. It started in New Zealand and ends around 24 hours later in the west coast of the USA.

As organisators, we were located roughly in the middle, the Netherlands, from which we wanted to keep track of what was happening and making sure the infrastructure was holding up.

This year, there were a lot of moving parts involved. A couple of days before the event, we had provisioned a large number of both Azure resources as well as AzureDevOps projects. My collegau Rob Bos has more details about this part on this [blog](https://rajbos.github.io/blog/2019/06/23/GDBC-Azure-learnings).

During the event itself, we had a couple of parts;

1. A challenges website which was used by the participants to read about the different challenges and to provide the ability to start and stop them.
2. Backend that was able to start containers which are used to control the disruption as well to influence the scoring.
3. A scoreboard to bring some competition into play.

Issues with any of these components could bring the user experience down or even made it hard for the people to do the challenges. Although we had communicated workaround for when elements were failing, it would be a shame if they could not be used as intended.

## Keeping track of users

We wanted to know what the users (or actually teams, we had accounts per team) were doing during the day. For this we used two different systems; Google Analytics and Application Insights. Adding Google Analytics is relatively easy. The code to add is provided by Google and can be added to the `_layout.cshtml` master layout like this

```csharp
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-141001163-1"></script>
    <script>
      window.dataLayer = window.dataLayer || [];
      function gtag() { dataLayer.push(arguments); }
      gtag('js', new Date());

      gtag('config', 'UA-141001163-1');
</script>
```

For Application Insights, you will need to do a similar action, by adding the code shown in [their documentation](https://docs.microsoft.com/en-us/azure/azure-monitor/app/javascript) to the master page.

The Google Analytics has some great dashboards that even allows you to see real time usage of your site and from where the request originated from. So at the start of the event, New Zealand was clearly the first to come online.

![newzealand.png](/images/newzealand.png)

It also allowed to see which pages are requested. Since we had the details of the challenge inside the url, we could see which ones were started.

[gdbcpages.png](/images/gdbcpages.png)

During the day, we nicely noticed the user location change along from Europe

![gdbceurope.png](/images/gdbceurope.png)

To the America continent

![gdbcamerica.png](/images/gdbcamerica.png)

Application Insight was also keeping track of the users which allowed us for example to make funnels to see if users not only looked at the challenges, but also started them and completed them.

![screen37.png](/images/screen37.png)

Also after the event itself, we could retrieve a lot of interesting information from both system. 




