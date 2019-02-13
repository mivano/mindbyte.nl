---
published: false
title: Retrieve the callback url of a logic app inside your ARM template
tags:
  - Azure
  - IaC
  - LogicApp
  - ARM
---
In a recent project, we use Infrastructure as Code to roll out resources to Azure. For this, we use ARM templates, and part of this template is the deployment of Azure Alerts and a Logic App. 
When an alert fires, the Logic App is called, which performs some processing and forwards the call to our incident management tool called [VictorOps](https://victorops.com/).

The alerts use a webhook, so it needs to know the address of the Logic App to deliver the payload to. The classic alerts allowed you to specify a webhook directly, the new style of alerts use Action Groups. Each action group contains one or more actions to perform like sending an email, or calling a logic app.

For example:

```json
"parameters": {
    "workflowName": {
      "type": "string"
    },
    "actionGroupName": {
      "type": "string"
    }    
  },
"resources":[   
{
      "type": "microsoft.insights/actionGroups",
      "name": "[parameters('actionGroupName')]",
      "apiVersion": "2018-03-01",
      "location": "Global",
      "tags": {},
      "scale": null,
      "properties": {
          "groupShortName": "[parameters('actionGroupName')]",
          "enabled": true,
          "emailReceivers": [],
          "smsReceivers": [],
          "webhookReceivers": [
            {
              "name": "logicapp_webhook",
              "serviceUri": "[listCallbackURL(concat(resourceId('Microsoft.Logic/workflows', parameters('workflowName')), '/triggers/manual'), '2016-06-01').value]"     
             }
          ],
          "itsmReceivers": [],
          "azureAppPushReceivers": [],
          "automationRunbookReceivers": [],
          "voiceReceivers": [],         
          "azureFunctionReceivers": []
      },
      "dependsOn": [
          "[resourceId('Microsoft.Logic/workflows', parameters('workflowName'))]"
      ]
    }]
```

In this example, we have a `serviceUri` defined pointing to the actual callback URL which is exposed by the Http action inside the logic app. We need to reference the `'/triggers/manual'`. If you do not do so, you reference the actual workflow definition itself which cannot be used to send data to.

So use `[listCallbackURL(concat(resourceId('Microsoft.Logic/workflows', parameters('workflowName')), '/triggers/manual'), '2016-06-01').value]` to fetch the actual callback url which allows you to post data to your Logic App.
