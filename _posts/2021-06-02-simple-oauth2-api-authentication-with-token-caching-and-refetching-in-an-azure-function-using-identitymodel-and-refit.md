---
published: true
featured: false
comments: false
title: Simple OAuth2 API authentication with token caching and refetching in an Azure Function using IdentityModel and Refit 
tags:
  - Coding
  - Azure
  - OAuth2
  - API
  - Functions
---

Connecting to an HTTP API is tricky enough, let alone handling the authentication to it. Many modern APIs allow you to provide an authentication key in the header, like the bearer token. You will need to fetch this token from a token provider, store it securely and handle its expiration. The token's lifetime is deliberately short, so you will need a way to fetch a new token. Retrieving it before every call you do to the API is also inefficient, so you need to manage this somehow.

For this, I usually fall back to the libraries of Dominick Baier and Brock Allen, and in this case, the `IdentityModel.AspNetCore` package. Their [documentation](https://identitymodel.readthedocs.io/en/latest/aspnetcore/overview.html) is pretty good, but let me walk you through an example where I recently added this to an Azure Function that needed to do a call to an API.

Make sure to add the package first from [NuGet](https://www.nuget.org/packages/IdentityModel.AspNetCore/). In this example, I also use [Refit](https://github.com/reactiveui/refit), but that is not a requirement.

In the `startup.cs`, I first register the access token management service. That will take care of a cache (in memory), a refresh mechanism (using Client Credentials flow), and adds the bearer token to all the calls to the service.

```csharp
builder.Services.AddAccessTokenManagement(options =>
  {
      options.Client.Clients.Add("api", new ClientCredentialsTokenRequest
      {
          RequestUri = new Uri(new Uri("https://api.com"), new Uri("/auth/token", UriKind.Relative)),
          ClientId = "client-id",
          ClientSecret = "client-secret"
      });
  });
```

The next step is to define an HTTP client implementation.

```csharp
builder.Services
  .AddRefitClient<IApiOperations>()
  .ConfigureHttpClient(client => client.BaseAddress = new Uri("https://api.com"))
  .AddClientAccessTokenHandler("api");
```

The `AddClientAccessTokenHandler` connects them together. The library will now handle the fetching of the token, caching, and refreshing with minimal coding.

The Refit library will create a nice wrapper for the API. In the interface you define the calls and their parameters.

```csharp
[Headers("Authorization: Bearer")]
public interface IApiOperations
{
    [Headers("Accept: application/json")]
    [Get("/business/by-external/{id}")]
    Task<ApiResponse<Business>> GetBusiness(string id);

    [Headers("Content-Type: application/json")]
    [Post("/business/")]
    Task<ApiResponse<Business>> PostBusiness([Body(BodySerializationMethod.Serialized)] Business business);
}
```

## Run in Azure Function

To call the API, you will need to inject the `IAPIOperations` instance into the constructor.

```csharp
public class PostBusinessActivity 
{
    private readonly IApiOperations _api;

    public PostBusinessActivity(
        IApiOperations api) 
    {
        _api = api;
    }

    [FunctionName(nameof(PostBusinessActivity))]
    public async Task<Business> RunAsync(
        [ActivityTrigger] Business business,
        ILogger logger)
    {
        var response = await _api.PostBusiness(business);
       
        return response.Content;
    }
}
```

Although the code above works in an Azure Function, it does enable the Authentication middleware as that is included in the `AddAccessTokenManagement` function. This will generate the following error:

```csharp
An unhandled host error has occurred.
Microsoft.AspNetCore.Authentication.Core: No authentication handlers are registered. Did you forget to call AddAuthentication().Add[SomeAuthHandler]("ArmToken",...)?.
```

The `/admin` routes are now secured, limiting certain tools (like VS Code) to access them and, in return, generating exceptions. 

To overcome this, I created a custom implementation of `AddAccessTokenManagement` with that line commented out.

```csharp
public static class AccessTokenManagementExtensions
{
    public static TokenManagementBuilder AddCustomAccessTokenManagement(this IServiceCollection services, Action<AccessTokenManagementOptions> options = null)
    {
        if (options != null)
        {
            services.Configure(options);
        }

        services.AddHttpContextAccessor();
        // services.AddAuthentication(); // Explicitly disabled as this interferes with the functions admin endpoint
        services.AddDistributedMemoryCache();

        services.TryAddTransient<IAccessTokenManagementService, AccessTokenManagementService>();
        services.TryAddTransient<ITokenClientConfigurationService, DefaultTokenClientConfigurationService>();
        services.TryAddTransient<ITokenEndpointService, TokenEndpointService>();

        services.AddHttpClient(AccessTokenManagementDefaults.BackChannelHttpClientName);

        services.AddTransient<UserAccessTokenHandler>();
        services.AddTransient<ClientAccessTokenHandler>();

        services.TryAddTransient<IUserTokenStore, AuthenticationSessionUserTokenStore>();
        services.TryAddTransient<IClientAccessTokenCache, ClientAccessTokenCache>();

        return new TokenManagementBuilder(services);
    }
}
```

I doubt that this is needed in an ASPNET webapp, but it was helped for an Azure function. 