---
published: false
featured: false
comments: false
title: Generate Kubernetes Access Token
description: >-
  How to retrieve an access token to use for the API calls to the Kubernetes
  cluster
tags:
  - Azure
  - AKS
---
There are various ways to connect to a Kubernetes cluster, but for a recent event ([GDBC](https://globaldevopsbootcamp.com/)) I needed to connect to an Azure Kubernetes Cluster using an API call. For this event we used Docker containers to do the actual work and the underlying infrastructure needed to schedule these containers to one of the available AKS instances. 