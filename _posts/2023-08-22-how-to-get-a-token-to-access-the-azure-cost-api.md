---
published: 2023-08-22T20:30:52.622Z
title: How to get a token to access the Azure cost API
tags:
  - Azure
  - Cost
header:
  teaser: 'https://mindbyte.nl/images/8-1vIMa336IHTO9Ks.png'
slug: token-access-azure-cost-api
---

The [Azure Cost CLI](https://github.com/mivano/azure-cost-cli) is a command-line dotnet tool created to facilitate interaction with Azure's cloud costs. While the development of the tool was a technical endeavor, it posed a significant challenge in one particular area: authentication.

## Authentication Complexity

Obtaining a token to access the Azure Cost API without burdening users with usernames, passwords, or the creation of service accounts was a challenging issue. The need for a straightforward application that could retrieve the account information directly from the environment necessitated a unique approach.

## Utilizing DefaultAzureCredentials

The answer was found in `DefaultAzureCredentials`. This functionality attempts [various credential providers](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/identity/Azure.Identity/src/Credentials/DefaultAzureCredential.cs) to find a valid one, offering a seamless way to authenticate. The approach enabled the retrieval of a token without adding complex authentication steps.

## ChainedTokenCredential with AzureCliCredential

To refine the process, a new `ChainedTokenCredential` was created, placing the `AzureCliCredential` at the start and then falling back to the default options if necessary. Below is the code snippet that encapsulates this elegant solution:

```csharp
// Get the token by using the DefaultAzureCredential, but try the AzureCliCredential first
var tokenCredential = new ChainedTokenCredential(
    new AzureCliCredential(),
    new DefaultAzureCredential());

// Fetch the token and ask explicitly for the Azure Cost API scope
var token = await tokenCredential.GetTokenAsync(new TokenRequestContext(new[]
    { $"https://management.azure.com/.default" }));

// Set the resulting token as the bearer token for the HTTP client
_client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token.Token);
```

### Simplicity and Flexibility

This method's success lies in its simplicity and flexibility, enhancing user experience and leveraging existing Azure CLI setups across different environments.

## Conclusion

The Azure Cost CLI tool is more than a utility; it represents a thoughtful response to a real-world problem. By innovatively combining `ChainedTokenCredential` with `AzureCliCredential`, a solution was crafted that is not only efficient and secure but also simple to use.

For developers and administrators seeking a user-friendly way to interact with the Azure Cost API, this approach provides a practical and elegant path forward.
