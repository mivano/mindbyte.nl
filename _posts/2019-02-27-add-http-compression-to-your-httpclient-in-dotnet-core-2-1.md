---
published: false
featured: false
comments: false
title: Add http compression to your httpclient in dotnet core 2.1
description: >-
  Use the HttpClientFactory to add compression to the dotnet core 2.1
  HttpClientHandler for making API calls.
categories:
  - http-api
tags:
  - api
  - dotnet
---
Microsoft added a nice feature in dotnet core 2.1 that allows you to register HttpClient instances in a central place and inject them using _Dependency Injection_ where you need them.

The HttpClientFactory manages the lifetime of the HttpClient for you and makes sure any additional middleware is executed. For example, you can add retry logic or logging globally. There are better [places](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-2.1) to describe what a HttpClientFactory does, here I want to discuss compression.

If you use your HTTP client to call another website, then by default it does not advertise that it supports compression. Which is a shame as it can reduce the payload of the response considerably. Most web servers will compress the response body when the request contains a header called `Accept-Encoding`. The value of this header specifies what kind of encodings are supported by the calling client. For example:

```
GET https://www.example.com
Accept: application/xml
Accept-Encoding: gzip, deflate
```

The server can respond with a compressed body:

```
200 OK
content-encoding: gzip
content-length: 17800
content-type: application/xml

<compressed>
```

The client now needs to decompress the body and return the contents. Your browser will by default do this for you, however a HttpClient won't.

## Configure a HttpClient

A simple configuration of a HttpClient takes place in the `ConfigureServices` function in your `startup` class. 

```csharp
services.AddHttpClient("nameofclient", client =>
            {
                client.BaseAddress = new Uri("http://www.httpbin.org");
                client.DefaultRequestHeaders.Add("Accept", "application/xml");
            })
```

Besides registering the HttpClient with a name and some default settings, you can add additional delegates like retry and exception handling. However, the compression cannot be directly configured as you need access to the HttpClientHandler instead.

```csharp
 services.AddHttpClient("nameofclient", client =>
            {
                client.BaseAddress = new Uri("http://www.httpbin.org");
                client.DefaultRequestHeaders.Add("Accept", "application/xml");
            })
 .ConfigurePrimaryHttpMessageHandler(messageHandler =>
            {
                var handler = new HttpClientHandler();

                if (handler.SupportsAutomaticDecompression)
                {
                   handler.AutomaticDecompression = DecompressionMethods.Deflate | DecompressionMethods.GZip;
                }
                return handler;
            });
```

You can configure a new HttpClientHandler, check if compression is indeed supported and select the encodings you want to use. The HttpClient will now send the header and will automatically decompress the response when encoded.

You can see the header by calling httpbin.org from within a controller:

```csharp
    [Route("api/[controller]")]
    [ApiController]
    public class ValuesController : ControllerBase
    {
        private IHttpClientFactory _clientFactory;

        public ValuesController(IHttpClientFactory clientFactory)
        {
            _clientFactory = clientFactory;
        }

        // GET api/values
        [HttpGet]
        public async Task<IActionResult> Get()
        {
            var client = _clientFactory.CreateClient("nameofclient");
            var result = await client.PostAsync("headers", new StringContent("")).ConfigureAwait(false);

            return Ok(result);
        }
     }
```

The `/headers` endpoint will echo back the supplied headers send to the endpoint.

## Conclusion

Compression can help in reducing the payload and thus speed up the transfer of data. However, a default HttpClient does not have this enabled. Using the HttpClientFactory, you can do this from a central location and reuse the logic.