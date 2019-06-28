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
Every year, Xpirit and Solidy organize the [Global DevOps Bootcamp](https://globaldevopsbootcamp.com/), an online event that teaches people about the DevOps practices using Microsoft technologies. As the name global  implies, the event takes place on the same day for around 100 venues all over the world. It started in New Zealand and ends around 24 hours later in the west coast of the USA.

As organisators, we were located roughly in the middle, the Netherlands, from which we wanted to keep track of what was happening and making sure the infrastructure was holding up.

This year, there were a lot of moving parts involved. A couple of days before the event, we had provisioned a large number of both Azure resources as well as AzureDevOps projects. My collegau Rob Bos has more details about this part on this [blog](https://rajbos.github.io/blog/2019/06/23/GDBC-Azure-learnings).

During the event itself, we had a couple of parts;

1. A challenges website which was used by the participants to read about the different challenges and to provide the ability to start and stop them.
2. Backend that was able to start containers which are used to control the disruption as well to influence the scoring.
3. A scoreboard to bring some competition into play.

Issues with any of these components could bring the user experience down or even made it hard for the people to do the challenges. Although we had communicated workaround for when elements were failing, it would be a shame if they could not be used as intended.

## Keeping track of users

We wanted to know what the users (or actually teams, we had accounts per team) were doing during the day. For this we used two different systems; Google Analytics and Application Insights. 
