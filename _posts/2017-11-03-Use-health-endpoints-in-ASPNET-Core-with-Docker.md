---
published: true
title: Use health endpoints in ASPNET Core with Docker
tags:
  - docker
header:
  image: /images/healthdocker.png
---

It is important to know if your nicely created application is still working correctly, but you do not want to keep refreshing your browser and seeing if that page is still returning the data you expect. Luckily there are better ways than just creating a specific endpoint to hit.

With some help of [App Metrics](https://www.app-metrics.io/) you can easily add one or more health checks to your application. App Metrics is for sure not the only application framework that can do this. Microsoft has a nice [implementation](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/implement-resilient-applications/monitor-app-health) too. App Metrics however also focuses on the metrics side (optionally) which adds some nice additional options to your application. It also has support for dotnet core and contains extensive documentation.

In general, they work the same; you define a set of health checks and by calling a specific endpoint you get the aggregated result back in the form of a status code and JSON representation. 

## Sample

To demonstrate this, you can use the samples provided by App Metrics from [GitHub](https://github.com/AppMetrics/Samples.V2/tree/master/AspNetCore2.Health.Api.QuickStart).

In 'program.cs' you will find the following code:

```csharp
  WebHost.CreateDefaultBuilder(args)
            .ConfigureHealthWithDefaults(
                builder =>
                {
                    const int threshold # 100;
                    builder.HealthChecks.AddCheck("DatabaseConnected", () => new ValueTask<HealthCheckResult>(HealthCheckResult.Healthy("Database Connection OK")));
                    builder.HealthChecks.AddProcessPrivateMemorySizeCheck("Private Memory Size", threshold);
                    builder.HealthChecks.AddProcessVirtualMemorySizeCheck("Virtual Memory Size", threshold);
                    builder.HealthChecks.AddProcessPhysicalMemoryCheck("Working Set", threshold);
                    builder.HealthChecks.AddPingCheck("google ping", "google.com", TimeSpan.FromSeconds(10));
                    builder.HealthChecks.AddHttpGetCheck("github", new Uri("https://github.com/"), TimeSpan.FromSeconds(10));
                })
                .UseHealth()
                .UseStartup<Startup>()
                .Build();
```

A couple of health checks are created and the UseHealth extension enables the functionality.

When you compile and run this code and go to the http://localhost:5000/health endpoint, you will get back something like this:

```json
{
  "healthy": {
    "DatabaseConnected": "Database Connection OK",
    "github": "OK. https://github.com/",
    "google ping": "OK. google.com"
  },
  "degraded": {
    "Sample Health Check": "DEGRADED"
  },
  "unhealthy": {
    "Private Memory Size": "FAILED. 114225152 > 100 bytes",
    "Virtual Memory Size": "FAILED. 2218805383168 > 100 bytes",
    "Working Set": "FAILED. 69939200 > 100"
  },
  "status": "Unhealthy"
}
```

As you can imagine, you can add any kind of check and by calling this from an API monitoring tool like [RunScope](https://www.runscope.com), your load balancer, [New Relic Synthetics](https://www.newrelic.com) you can check the state of your application and act accordingly.

## Docker

So how is Docker able to use this? Since version 1.12 there is support for [HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#healthcheck). You can specify an instruction for Docker to use to validate the application running inside the container. It also exposes a health status next to the normal status of a container which can be queried using 'docker inspect' and is visible when using 'docker ps'.

For the above example you can use a DOCKERFILE containing the HEALTHCHECK instruction:

```
FROM microsoft/aspnetcore:2.0
ARG source
WORKDIR /app
HEALTHCHECK --interval=2s --timeout=3s --retries=1 CMD curl --silent --fail http://localhost:80/health || exit 1
EXPOSE 80
COPY ${source:-obj/Docker/publish} .
ENTRYPOINT ["dotnet", "AspNetCore2.Health.Api.QuickStart.dll"]
```

The interval, timeout, and retries are optional, the CMD will tell what the actual check is. In this case, it is a curl command to the /health endpoint. Since it returns a non 200 code when there is an unhealthy state, the curl command will exit with a 1. Do keep in mind that curl needs to be included in the container image and this might not always be the [best option](https://blog.sixeyed.com/docker-healthchecks-why-not-to-use-curl-or-iwr/).

![](/images/healthdocker.png)

Next to showing the health state, it will also raise an event which can be used by an orchestration engine to stop sending traffic to an unhealthy container instance and restart containers.

The healthcheck can also be set in a docker-compose.yml file

```yaml
healthcheck:
  test: curl --silent --fail http://localhost:80/health || exit 1
  interval: 5s
  timeout: 10s
  retries: 3
```  

## Swarm

When using Docker Swarm, you can also use the healthcheck options directly when creating the service by specifying the options health-cmd, health-retries and/or health-interval. When it detects an unhealthy container, it will restart the container automatically.

## Kubernetes

In Kubernetes it works a little bit different. You still use the /health endpoint, but specify a [livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-http-request). This will make sure that Kubernetes automatically restarts working containers having failed applications inside.
