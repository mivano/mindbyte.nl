---
published: false
featured: false
comments: false
title: Being on call
description: Being on call
tags:
  - oncall
  - devops
  - support
---
True DevOps means you build it, you run it. You are responsible for the whole stack, including production. And when it comes to production, you have of course the most stable system ever. Unfortunately issues can come up and it is very much likely that you need to be on call. 

In a recent project I needed to do the same; support a critical application that when having outages would impact the business.

Some learnings from this time I want to share:

1. Focus on the right telemetry and automatic monitoring. This should alert you, not your end users. If you get calls from them, then you are already too late. This is also something that needs constant adjusting to avoid false positives, but as part of the DevOps team you can do this.

1. Store logs in a centralized place, like using tools such as Application Insights or the ELK stack. You quickly want to see what the issue is and start throubleshooting. Dashboards sounds nice, but you would hardly look at those unless it is an issue.

1. Have runbooks available; the more you can prepare, the better. Runbooks describe the symptoms, the possible checks, resolution(s) and post steps. Have a logic index or search and even better, automatically link them to the alert.

1. Make use of an incident management tool; for example [Pagerduty](https://www.pagerduty.com/) or [VictorOps](https://victorops.com/). They provide an entry point for the alert and escalate to the right person or group. No hassle with phone numbers that do not work or point to the wrong person who has changed their shift. These tools allow escalation rules so issues can be picked up by others when the response is not fast enough. 

1. When an incident is raised, make sure to acknowlegde this first. So others are aware that somebody is looking at the issue. Tools mentioned above will do this for you. Assess the issue, see what is impacted and mitigate. In the middle of the night you wont have time to do a root cause analysis or write that nice solution, that is for the next day. Always perform a post mortem; why did the issue occur, was it a valid one to wake you up and what did you do to make sure it will not happen again.

1.
