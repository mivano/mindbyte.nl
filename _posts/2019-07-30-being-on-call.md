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
True DevOps means you build it, you run it. You are responsible for the whole stack, including production. And when it comes to production, you have of course the most stable system ever. Unfortunately, issues can come up and it is very much likely that you need to be on call. 

In a recent project, I needed to do the same; support a critical application that when having outages would impact the business.

Some learnings from this time I want to share:

1. Focus on the right telemetry and automatic monitoring. These should alert you, not your end-users. If you get calls from them, then you are already too late. This is also something that needs constant adjusting to avoid false positives, but as part of the DevOps team you can do this.

1. Store logs in a centralized place, like using tools such as Application Insights or the ELK stack. You quickly want to see what the issue is and start troubleshooting. A dashboard sounds nice, but you would hardly look at those unless it is an issue.

1. Have runbooks available; the more you can prepare, the better. Runbooks describe the symptoms, the possible checks, resolution(s) and post steps. Have a logic index or search and even better, automatically link them to the alert when they come in. When woken up in the middle of the night it is beneficial to follow a list of steps.

1. Make use of an incident management tool; for example [Pagerduty](https://www.pagerduty.com/) or [VictorOps](https://victorops.com/). They provide an entry point for the alert and escalate to the right person or group. No hassle with phone numbers that do not work or point to the wrong person who has changed their shift. These tools allow escalation rules so issues can be picked up by others when the response is not fast enough. 

1. When an incident is raised, make sure to acknowledge this first. So others are aware that somebody is looking at the issue. The tools mentioned above will do this for you. Assess the problem, see what is impacted and mitigate. In the middle of the night, you won't have time to do a root cause analysis or write that beautiful solution, that is for the next day. Always perform a post mortem; why did the issue occur, was it a valid one to wake you up and what did you do to make sure it will not happen again.

1. Communication is essential; make sure you can escalate and have a communication plan. The last thing you want to chasing others or get asked multiple times what the status is. We used [StatusPage.io](https://statuspage.io) to publish any issues we had with the application and in which state of the incident we were.

1. The on-call should not be longer than one week. It has an impact on your life and you need to relax from it after a week. This also means that there needs to be at least 4 or 5 people on the on call group to cover a month. More is better as there are also holidays etc. Tools like mentioned before allow for an easy take over of shifts, so it is possible to go out while somebody else covers your shift.

1. Rotate on a Tuesday (Mondays are more likely to be a holiday) and do this from the afternoon, so you had time for handovers.

1. Make sure that being on call is compensated. Either in time or money. Acting on an incident should be an additional fee. 

1. The on-call is for support, not for running out of business hour tasks. 

1. If there is a need for a response time shorter than like 20 minutes, then having an overnight shift or a follow the sun setup might be more useful. 

1. During business hours, have juniors or new joiners participate in the on-call so they can learn the tricks.

And remember; any alert that reached the on-call person should be something that warrants somebody getting out of bed and fix the issue right away. Everything else should wait for regular business hours.

Not having control over the product and as such, not doing DevOps would be a no go for me in providing on-call. If I'm not able to understand and fix the issues, then it is just a simple helpdesk function and adds little benefit.

