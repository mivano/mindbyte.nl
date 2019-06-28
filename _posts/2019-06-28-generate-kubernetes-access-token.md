---
published: true
featured: false
comments: false
title: Generate Kubernetes Access Token
description: >-
  How to retrieve an access token to use for the API calls to the Kubernetes
  cluster
tags:
  - Azure
  - AKS
header:
  '-image': images/silas-kohler-C1P4wHhQbjM-unsplash.jpg
---
There are various ways to connect to a Kubernetes cluster, but for a recent event ([GDBC](https://globaldevopsbootcamp.com/)) I needed to connect to an Azure Kubernetes Cluster using an API call. For this event, we used Docker containers to do the actual work and the underlying infrastructure needed to schedule these containers to one of the available AKS instances. 

## Connecting to the cluster

To talk to the AKS cluster, I used the open source [KubernetesClient](https://www.nuget.org/packages/KubernetesClient/) NuGet package. Under the hood, it will call the REST API of the cluster to perform actions.

Creating a connection is a matter of setting up a `KubernetesClientConfiguration` instance. 

```csharp
  var config = new KubernetesClientConfiguration
                    {
                        Host = clusterSetting.Host,
                        SkipTlsVerify = true,
                        AccessToken = clusterSetting.AccessToken
                    };
```

Here I used the hostname as found in the AKS blade in Azure and the access token that I retrieved as shown below. When I have a configuration, I can connect to the cluster itself and perform actions:

```csharp
 using (var client = new Kubernetes(config))
 {
  var response = await client.CreateNamespacedPodWithHttpMessagesAsync(
                            new V1Pod(
                                metadata: new V1ObjectMeta(name: message.MessageId,
                                    namespaceProperty: "default"
                                    , labels: new Dictionary<string, string>
                                    {                                      
                                        {"messageId", message.MessageId}
                                    }
                                ),
                                spec: new V1PodSpec(containers: messageData.Containers.Select((a, index) =>
                                        new V1Container
                                        {
                                            Name = $"{messageData.ChallengeId.ToLowerInvariant()}-{index + 1:D2}",
                                            Image = a.Name,
                                            ImagePullPolicy = "Always"                                                                                  }).ToList(),
                                    terminationGracePeriodSeconds: 0,
                                    restartPolicy: "Never")),
                            "default", cancellationToken: token).ConfigureAwait(false);
 }
```

The above code will create a Pod in the `default` namespace and starts one or more containers inside this Pod.

## Getting the Access Token

In order to get an access token, I need to create an account first and give it the correct rights.

Assuming we have connected to the cluster in some form, we can issue commands using the kubectl tool.

Lets first create a service account called `gdbc`.

```
kubectl create serviceaccount gdbc
```

Next, give this account access to the cluster:

```
kubectl create clusterrolebinding gdbc \
  --clusterrole=cluster-admin \
  --serviceaccount=default:gdbc
```

I can now find the secrets created and get the contents of the secret as it not such a secret that it is encrypted.

```
kubectl get secrets
```

This will return a list of secrets similar to:

```
NAME                  TYPE                                  DATA      AGE
default-token-t6j62   kubernetes.io/service-account-token   3         163d
gdbc-token-hhp2g      kubernetes.io/service-account-token   3         11s
```

Now fetch the one that starts with the provided name:

```
kubectl describe secret gdbc-token-hhp2g
```

The output will be something like this:

```
Name:         gdbc-token-hhp2g
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=gdbc
              kubernetes.io/service-account.uid=50266d42-9989-11e9-9728-00155df43454

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZheddeVzLmlvL3NlcweY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZ2RiYyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmdeS1hY2NvdW50LnVpZCI6IjUwMjY2ZDQyLTk5ODktMTFlOS05NzI4LTAwMTU1ZGY0MzQ1NCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmdkYmMifQ.Bd-inWYl4Y-wECjFkF7kEt0S-o3Ah17osJg5JGkAB2Xu7ypFS6ur6S6p6JiqjP9X9gsLfBBjXZrZn8RnOxUfuEoPFtjIFCnXI4xNsNWzSo8ahbwETb5lfGkMFFurYyxlSYqE3FO1X3L7xGQfYXaD5vEVZ01dTrZdQ87PH-lYhvwVmkIu3gmW6aLgEIapnU26PSkzv8Ca_q-3MMBSikkYrTB7GugTKRtHkgYWdN3NQXTmEBBqmmLHJKewtwacKiATpgp7J6_aMyiuY4arCvL8F0DYL3-YkuPfhpgTMEHDRuZdfQSsf8aZgTLaD3lP1MgxNqZBfWfIjO7vd2WAlvQmvQ
ca.crt:     1025 bytes
namespace:  7 bytes
```

The token is nothing more then a Json Web Token. This value can be used as the access token in the above API call.

## Conclusion

Getting the access token to make authenticated calls to a Kubernetes cluster is not that difficult, but be aware that you need to carefully store the access token and give it the correct amount of privileges. !
