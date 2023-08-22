---
published: 2023-08-25T22:59:51.000Z
title: Retrieving the active Azure Subscription ID from the AZ CLI Context
tags: []
header:
  teaser: 'https://mindbyte.nl/images/8-owJfv4h3t9ZHOaq.png'
slug: retrieving-active-azure-subscription-id-az-cli-context
---

Managing Azure subscriptions can be a delicate task, especially when dealing with various tools and applications that require context-specific information. In building the [Azure Cost CLI](https://github.com/mivano/azure-cost-cli) a requirement arose to determine the current active subscription ID when it's not explicitly passed in. This post will dive into the process of retrieving this information, similar to the Azure CLI's default behavior set by `az account set -s`.

## Default Subscription ID

The Azure CLI allows users to set a default subscription ID, and mimicking this behavior in the Azure Cost CLI tool brings in alignment and familiarity. The question was, how could this be achieved programmatically?

## Executing the AZ Command

The solution lay in executing the Azure CLI's `az account show` command, which returns information about the current account, including the active subscription ID.

The following C# code is used to execute the command and extract the subscription ID:

```csharp
public static class AzCommand
{
    public static string GetDefaultAzureSubscriptionId()
    {
        var startInfo = new ProcessStartInfo
        {
            FileName = "az",
            Arguments = "account show",
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true
        };

        using (var process = new Process { StartInfo = startInfo })
        {
            process.Start();
            string output = process.StandardOutput.ReadToEnd();
            process.WaitForExit();

            if (process.ExitCode != 0)
            {
                string error = process.StandardError.ReadToEnd();
                throw new Exception($"Error executing 'az account show': {error}");
            }

            using (var jsonDocument = JsonDocument.Parse(output))
            {
                JsonElement root = jsonDocument.RootElement;
                if (root.TryGetProperty("id", out JsonElement idElement))
                {
                    string subscriptionId = idElement.GetString();
                    return subscriptionId;
                }
                else
                {
                    throw new Exception("Unable to find the 'id' property in the JSON output.");
                }
            }
        }
    }
}
```

Here's what the code does, step by step:

1. **Creating Process Start Information**: A `ProcessStartInfo` object is initialized with the required command and settings.
2. **Executing the Process**: A new process is started to execute the command, and the standard output is read.
3. **Error Handling**: If the command execution fails, an error is thrown with the details.
4. **Parsing the Output**: The JSON output is parsed to retrieve the `id` property, which holds the subscription ID.

## Seamless Integration

The above code snippet integrates seamlessly into the Azure Cost CLI tool, providing an efficient and consistent way to retrieve the current active Azure subscription ID. By leveraging existing CLI commands and C# process management, developers can easily obtain context-specific information.

This example further illustrates how familiar tools and libraries can be used to build sophisticated features, bridging gaps between different systems and providing a cohesive user experience.
