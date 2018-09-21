---
published: true
title: Sending Deliver information to VictorOps from Azure DevOps releases using Bash
tags:
  - VSTS
  - AzureDevOps
  - IncidentManagement
header:
  image: images/patrick-hendry-613598-unsplash.png
---

When you are using an Incident Management alerting tool like Pagerduty or VictorOps, it might be beneficial to notice when your deployment system is performing an action. Like doing a release to your environment. Pretty useful to correlate with an incident coming in.

VictorOps has an integration called [Delivery Insights](https://help.victorops.com/knowledge-base/victorops-generic-delivery-insights-integration/). Basically a specific endpoint where third-party tooling can send a specific payload to so it can record a release.

VictorOps has GitHub integration, but their Azure DevOps (VSTS) integration is lacking. Luckily we can do this ourself. 

## Bash task

In your release definition, add a **Bash** task to your release tasks. As we use here a Linux agent, we need to use a bash script. For this, we will use the _inline_ script option. Give it a sensible name like `Tell VictorOps about the deployment`.

Inside the script paste the following:

```bash
payload='{
    "entity_type": "build system",
    "event_type": "Build",
    "source": "Azure DevOps",
    "summary": "'"$definition"'",
    "url": "'"$url"'",
    "action": "deployed",
    "result": "SUCCESS",
    "long_message": "Deployed to '"$environment"'"
}'

echo $payload
echo $victoropsurl

curl -H "Content-Type: application/json; charset=UTF-8" -X POST --data "$payload" $victoropsurl
```

This will construct a payload based on the fields documented by [VictorOps](https://help.victorops.com/knowledge-base/victorops-generic-delivery-insights-integration/). Of course you can alter the fields to your liking. Be careful about the escaping of the variables. 

## Variables
The VictorOps delivery endpoint URL we will put in the variables of the release. The value you can get from the VictorOps integration settings page. The other variables, used inside the payload, you first need to map using environment variables before you can consume them.

In the task, you can enter **Environment Variables**. This allows you to get system and user variables inside your script.

![victoropsbashtask.png](/images/victoropsbashtask.png)

## End result

When the release has been completed and your last task was this bash command, you will have a Deployment notification inside your timeline.

![victoropstimeline.png](/images/victoropstimeline.png)
