---
published: true
title: How to allow teams to use API Management while staying secure and compliant
tags:
  - API
  - APIM
  - REST  
---

Are you working with Azure Api Management in your organization? Do you let your feature teams deploy APIs from their pipelines? If so, how do you make sure they are compliant and secure? 

Here are some suggestions:

Have a central repository where you control the rollout of APIs. Guard this repo with pull requests and strict reviews. There is a big drawback; it is limiting the process and thus the independence of feature teams.

Another option is to scan frequently the API Management instance using the REST APIs and detect incorrect config. You do need to build the logic for the validation yourself and can be reactive. This is a lot of work, but can be done using the endpoints offered by APIM.

The last options is to use Azure Policies. Microsoft provides a set of policies to audit, correct or reject incorrect setups. https://docs.microsoft.com/en-us/azure/api-management/policy-reference 

So what do you need to check for? 

- Use of secure channels; HTTPS/WSS 
- Access to backend should not be bypassed 
- Secrets are placed in Keyvaults 
- Subscriptions are correctly scoped

Api Management is a powerful product and can easily be used by multiple teams by allowing them to rollout their definitions. Trusting them is good, but validation is better. So in that case use policies to help you with this.

