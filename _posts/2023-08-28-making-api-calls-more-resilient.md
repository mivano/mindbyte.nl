---
published: 2023-08-27T22:25:46.000Z
title: Making API calls more resilient
tags:
  - Azure
  - REST
  - API
header:
  teaser: 'https://mindbyte.nl/images/8-nMUDd7QJ0Y43Pf1.png'
slug: making-api-calls-resilient
---

In the world of networked applications, call failures are a common occurrence due to the inherent unreliability of networks. When working with the Azure Cost API, challenges such as indefinite retries, excessive wait times, and rate limitations can impede efficiency and efficacy. This post explores a method to make API calls more resilient, taking into account these critical factors.

## Unreliable Network and Rate Limitations

When making requests to an API, calls can fail for various reasons ranging from network unreliability to server-side rate limits. Repeated retries, lengthy waiting periods, or disregard for server-specified limitations can lead to inefficiencies or even service denial.

## Polly - .NET Resilience Library

To address these challenges, [Polly](https://www.thepollyproject.org), a .NET resilience and transient-fault-handling library, was employed. It enables developers to define policies like Retry, Circuit Breaker, Timeout, Bulkhead Isolation, and Fallback, thereby increasing the resilience of API calls.

## Defining a Resilient Policy with Polly

The core of this approach lies in defining a resilient policy that combines a wait-and-retry policy with a timeout policy. Below is the code snippet that showcases this process:

```csharp
public static class PollyPolicyExtensions
{
    public static IAsyncPolicy<HttpResponseMessage> GetRetryAfterPolicy()
    {
        // Define WaitAndRetry policy
        var waitAndRetryPolicy = Policy.HandleResult<HttpResponseMessage>(msg => 
                msg.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(
                retryCount: 5,
                sleepDurationProvider: (_, response, _) => {
                    var headers = response.Result?.Headers;
                    if (headers != null)
                    {
                        foreach (var header in headers)
                        {
                            if (header.Key.ToLower().Contains("retry-after") && header.Value != null)
                            {
                                if (int.TryParse(header.Value.First(), out int seconds))
                                {
                                    return TimeSpan.FromSeconds(seconds);
                                }
                            }
                        }
                    }
                    // If no header with a retry-after value is found, fall back to 2 seconds.
                    return TimeSpan.FromSeconds(2);
                },
                onRetryAsync: (msg, time, retries, context) => Task.CompletedTask
            );

        // Define Timeout policy
        var timeoutPolicy = Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(60));

        // Wrap WaitAndRetry with Timeout
        var resilientPolicy = Policy.WrapAsync(timeoutPolicy, waitAndRetryPolicy);

        return resilientPolicy;
    }
}
```

This policy intelligently handles HTTP 429 (Too Many Requests) responses by parsing the "retry-after" header and waiting for the specified duration before retrying, up to five times. If the header isn't found, it falls back to a two-second wait time. Additionally, a timeout policy ensures that calls do not wait longer than 60 seconds.

## Applying the Policy

The defined policy is then applied to the HTTP client, as shown below:

```csharp
registrations.AddHttpClient("CostApi", client =>
{
  client.BaseAddress = new Uri("https://management.azure.com/");
  client.DefaultRequestHeaders.Add("Accept", "application/json");
}).AddPolicyHandler(PollyPolicyExtensions.GetRetryAfterPolicy());
```

This registration is placed in the `ConfigureServices` method of the `Startup` class, ensuring that the policy is applied to all HTTP calls made by the application when they request this `CostApi` httpclient from the factory.

## Conclusion

The combination of Polly's features offers a robust and scalable solution to the common challenges faced when making resilient API calls. By leveraging this method, developers can create a more stable and efficient communication with the Azure Cost API, providing a seamless experience even in unpredictable network conditions.

For those looking to enhance their API interactions, exploring Polly and the resilient policies it supports can be a game-changer. The flexibility and reliability it offers make it an essential tool for modern developers navigating the complexities of network communication.
