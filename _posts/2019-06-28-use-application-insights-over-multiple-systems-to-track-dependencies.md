---
published: true
featured: false
comments: false
title: Use Application Insights over multiple systems to track dependencies
description: Use Application Insights over multiple systems to track dependencies
tags:
  - azure
  - dotnet
  - applicationinsights
  - gdbc
---
It is essential to have an overview of what your software is doing, not only to troubleshoot issues but also to see how data flows through your system and how it is used. 
For a recent global event ([GDBC](https://globaldevopsbootcamp.com/)), we had a couple of different components that needed to interact with each other. There was a website where people could start challenges and a backend that scheduled those challenges. Connecting these together was an Azure Servicebus.

## Trace over dependency boundaries

In the website, there will be a message queued onto the servicebus. What we want is to trace that somebody initiated this request, that it is indeed queued and that it will be retrieved on the other side. For this to happen, we need to pass in some sort of correlation id. We inject a `telemetryClient` into the class and wrap the call to the servicebus like shown below:

```csharp
var telemetry = new DependencyTelemetry
{
     Type = GetType().Name,
    Name = "Save to queue"
};
telemetry.Start();

telemetry.Context.Operation.Id = _telemetryClient?.Context?.Operation?.Id;
telemetry.Context.Operation.ParentId = _telemetryClient?.Context?.Operation?.ParentId;

using (var operation = _telemetryClient.WrapTelemetry(telemetry))
{
  operation.SetType("ServiceBusSender");
  operation.SetData($"Enqueue {_queueName}");

  try
  {
    // Create a new message to send to the queue.
    var messageBody = JsonConvert.SerializeObject(payload);
    var brokeredMessage = new Message(Encoding.UTF8.GetBytes(messageBody));

    // Write the body of the message to the console.
    _logger.LogTrace("Sending message: {messageBody}", messageBody);

    string operationId = operation.GetOperationId();
    var telemetryId = operation.GetTelemetryId();
    if (operationId != null) brokeredMessage.UserProperties["RootId"] = operationId;
    if (telemetryId != null) brokeredMessage.UserProperties["ParentId"] = telemetryId;

    // Send the message to the queue.
    await queueClient.SendAsync(brokeredMessage);

    operation.SetSuccess(true);
  }
  catch (Exception exception)
  {
    _logger.LogError(exception, "Unable to send message to the servicebus. Trying to start {ContainerName}: {ExceptionMessage}", payload.Containers.Select(a => a.Name), exception.Message);

    operation.SetSuccess(false);
  }
}
```

Inside the `brokeredMessage` we store the root and parent id's which allows us to keep the trace intact. Of course, we need to extract those on the other side:

```csharp
async Task ProcessMessagesAsync(Message message, CancellationToken token)
{

  var telemetry = new DependencyTelemetry
  {
    Type = GetType().Name,
    Name = "Read from queue"
    };
  message.UserProperties.TryGetValue("RootId", out var operationId);
  message.UserProperties.TryGetValue("ParentId", out var parentOperationId);

  telemetry.Start();

  telemetry.Context.Operation.Id = operationId?.ToString();
  telemetry.Context.Operation.ParentId = parentOperationId?.ToString();

  telemetry.Start();

  using (var operation = _telemetryClient.StartOperation<DependencyTelemetry>(telemetry))
  {
    operation.Telemetry.Type = nameof(ServiceBusSender);

    try
    {
      await _challengesRunner.ProcessMessage(message, token).ConfigureAwait(false);
    }
    catch (Exception exception)
    {
      _logger.LogError(exception,
                       $"Unable to process message from the servicebus: {exception.Message}");

      operation.Telemetry.Success = false;
    }
    finally
    {
      // Complete the message so that it is not received again.
      // This can be done only if the queue Client is created in ReceiveMode.PeekLock mode (which is the default).
      await _queueClient.CompleteAsync(message.SystemProperties.LockToken);

      // Note: Use the cancellationToken passed as necessary to determine if the queueClient has already been closed.
      // If queueClient has already been closed, you can choose to not call CompleteAsync() or AbandonAsync() etc.
      // to avoid unnecessary exceptions.
      _telemetryClient.StopOperation(operation);
    }
  }
}
```

The `TryGetValue` functions will try to get the id's from the message and when found, add them to the trace.

## Application Insights

The end result in Application Insight is as follows:

![screen7.png](/images/screen7.png)

As you can see, a challenge is started, the message is placed on the queue and picked up again by the backend. The backend on its turn will use the Kubernetes runner to start the challenge itself.

![screen1.png](/images/screen1.png)

Also, in the application map, you can see the dependencies. There is a directed graph from the challenges website to the control plane component with the number of calls.

## Conclusion
Using Application Insights is a powerful way to see the dependencies between components, but you do need to do some work for it. Luckily this is relatively simple by wrapping the calls using the Telemetry client.
