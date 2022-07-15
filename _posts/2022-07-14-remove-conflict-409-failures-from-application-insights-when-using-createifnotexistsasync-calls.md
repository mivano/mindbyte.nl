---
published: true
title: Remove conflict (409) failures from Application Insights when using CreateIfNotExistsAsync calls
tags:
  - Azure
  - ApplicationInsights  
  - Logging
---

Application Insights is not only an excellent tool to see what your application is doing, but it also shows you how your dependencies are behaving. There is not much that you need to configure to get this feature as our of-the-box Application Insights [tracks a number of dependencies automatically](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-dependencies). If this is not enough and you need to track another dependency yourself, you can add [DependencyTelemetry to your code](https://mindbyte.nl/2019/06/28/use-application-insights-over-multiple-systems-to-track-dependencies.html).

One of the dependency telemetry features is to indicate if a call was successful or not by adding a `call status` of false. As such, they will appear in the Failed requests section of the Application Insights dashboard.

Unfortunately, this is also the case if you perform a check to validate if a container or queue exists. For example, using this method:

```csharp
await table.CreateIfNotExistsAsync()
```

This [method](https://docs.microsoft.com/en-us/dotnet/api/azure.storage.blobs.blobcontainerclient.createifnotexistsasync) tries to create a container or queue and will get a 409 (a conflict) if this item already exists. However, the 409 code is considered an error, and the dependency telemetry will show this as a failed request. A check if something exists will also throw a 404 code, which is considered an error as well.

This does not provide an accurate representation of what is happening, so here are a couple of suggestions:

Create queues and containers in a separate phase instead of each time. If you have static known names for your queues and containers, you can create them at your application's startup or during deployment.

Another option is to simply mark them as successful. You can do this with an `ITelemetryInitializer` by changing the success to be true.

```csharp
public class AzureCheckExistsInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        if (telemetry is DependencyTelemetry dependencyTelemetry &&
            (dependencyTelemetry.Type == "Azure table" || dependencyTelemetry.Type == "Azure blob") &&
            dependencyTelemetry.ResultCode == "409" &&
            dependencyTelemetry.Success == false)
        {
            dependencyTelemetry.Success = true;
        }
    }
}
```

Register this initializer in the startup logic of your application:

```csharp
services.AddSingleton<ITelemetryInitializer, AzureCheckExistsInitializer>();
```

However, keeping this kind of telemetry around might also not be a good idea. Although Application Insight will perform sampling, dependency tracking can consume a lot of data and cost. You can use an `ITelemetryProcessor` to filter out the telemetry you don't want to see.

```csharp
public class RemoveConflictDependencyFilter : ITelemetryProcessor
{
    private ITelemetryProcessor Next { get; set; }

    // next will point to the next TelemetryProcessor in the chain.
    public RemoveConflictDependencyFilter(ITelemetryProcessor next)
    {
        this.Next = next;
    }

    public void Process(ITelemetry item)
    {
        // To filter out an item, return without calling the next processor.
        if (IsConflict(item)) { return; }

        this.Next.Process(item);
    }
    
    private bool IsConflict(ITelemetry item)
    {
        return item is DependencyTelemetry dependencyTelemetry &&
                (dependencyTelemetry.Type == "Azure table" || dependencyTelemetry.Type == "Azure blob") &&
                dependencyTelemetry.ResultCode == "409" ;
    }
}
```

Again, register this processor in the startup logic of your application:

```csharp
services.AddApplicationInsightsTelemetryProcessor<RemoveConflictDependencyFilter>();
```

So dependencies can be changed or filtered out or be explicit in the calls you make to dependencies. Even if you change or filter, the actual network call will still be made and might not always be needed.


