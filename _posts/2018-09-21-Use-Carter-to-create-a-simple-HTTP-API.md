---
published: false
title: Use Carter to create a simple HTTP API
category:
  - HTTP-APIs
tags:
  - api
  - http
categories:
  - HTTP-APIs
---
Creating a HTTP API can be done in multiple ways, but if you are .NET developer, you most likely will use the [ASP.NET Web API framework](https://www.asp.net/web-api). In previous versions this was not that obvious to use. The MVC framework and the Web API framework were two different systems. Action filters for one, did not work for the other framework. With newer versions this difference was solved by having one integrated system.

However there are other frameworks that allow you to do similar things. It is all a matter of taste what works best for you, your team and the project. In this post I will show you Carter.

## Carter

[Carter](https://github.com/CarterCommunity/Carter) is a ASP.NET Core library to that adds Nancy like routing support to your project. It works on top of the default Microsoft routing to provide a more elegant like routing syntax. If you ever worked with NancyFX you will recognize it directly, but I will show it in some examples. It is created by [Jonathan Channon](https://github.com/jchannon) who is also a contributor to NancyFx.

## Installing

We start by creating a new **ASP.NET Core Web Application** and make sure it is an empty project. Adding Carter is as simple as adding the [NuGet](https://www.nuget.org/packages/Carter/) package:

```powershell
Install-Package Carter
```

After installing some dependencies (like the nice FluentValidator and some MS AspNet core packages) you need to setup the pluming part of it. Inside the **startup.cs** file we need to let the system know that we want to use Carter:

```csharp
  public class Startup
    {
        // This method gets called by the runtime. Use this method to add services to the container.
        // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddCarter();   // <-- add this line
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseCarter(); // <-- add this line
        }
    }
```

## Creating your first module

Carter will search **CarterModule**s implementations, so lets create one:

```csharp
 public class HomeModule : CarterModule
 {
        public HomeModule() : base("api")
        {
            Get("/", async (req, res, routeData) =>
            {
                await res.AsJson(req.Headers);
               
            });
            
        }
 }
 ```
    
This _HomeModule_ lives under the `/api` root, as that is what we specified inside the call to the base class. Inside the constructor, we defined the different methods this module will offer. In this case a simple `GET` method. 

As parameters you have the request, response and route data which allows you to do something with the incoming data and construct a response. In this case the call will return a JSON collection of the request headers.

You can find some more [samples at GitHub](https://github.com/CarterCommunity/Carter/tree/master/samples).

## Handling data




