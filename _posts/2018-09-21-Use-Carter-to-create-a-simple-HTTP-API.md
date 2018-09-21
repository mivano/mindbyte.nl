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

Carter supports model binding, so mapping the incoming data to a model. Lets create a simple order which we are updating.

```csharp
 public class OrdersModule : CarterModule
    {

        public OrdersModule(): base("orders")
        {
            
            Put("{orderId:int}", async  (req, res, routeData) =>
            {
                var result = req.BindAndValidate<Order>();

                if (!result.ValidationResult.IsValid)
                {
                    res.StatusCode = 422;
                    await res.Negotiate(result.ValidationResult.GetFormattedErrors());
                    return;
                }

                // TODO Update in DB

                res.StatusCode = 200;
                await res.Negotiate(result.Data);
            });

        }
    }

    public class Order
    {
        public int Id { get; set; }
        public decimal Price { get; set; }

    }

    public class OrderValidator : AbstractValidator<Order>
    {
        public OrderValidator()
        {
            this.RuleFor(x => x.Price).GreaterThan(0);
        }
    }

```

Here we have an **OrdersModule**, living under the `/orders` endpoint. A `PUT` verb is defined and it expects an integer as identifier. The `BindAndValidate` function will map the contents of the form body to the object (in this case the `Order`) and validates. We then either return an error or perform an update in the database and return a 200 and the contents.

For the validation, we have the `OrderValidator`. Using the FluentValidator library we can specify the rules that this model need to apply to.

A `POST` and a `PATCH` work in a similar way.

As the system uses the default dependency injection, you can pass in any dependencies via the contructor. So a repository can be added like this:

'''csharp
  private readonly IOrderRepository _orderRepository;

        public OrdersModule(IOrderRepository orderRepository) : base("orders")
        {
```

Make sure to register this in the **startup.cs**.

```csharp
 services.AddTransient<IOrderRepository, DummyOrderRepository>();
```

## Authentication

When you want to secure your endpoints, you normally add the `AuthorizeAttribute` to your controller or define default policies. In Carter, you have a different way to handle this as it supports before and after hooks. Security is a typical thing to handle in the before hook, while the after hook is useful for logging etc.

First we create a module we want to secure:

```csharp
 public class SecureModule : CarterModule
    {
        public SecureModule() : base("secure")
        {
            Before += context =>
            {
                var authenticated = context?.User?.Identity != null && context.User.Identity.IsAuthenticated;
                if (!authenticated)
                {
                    context.Response.StatusCode = 401;
                }
                return Task.FromResult(authenticated);
            };

            Get("/", async (req, res, routeData) =>
            {
                await res.AsJson(req.HttpContext.User.Claims.Select(c=>new {c.Type, c.Value}));
            });
         
        }
    }
 ```
 
We use the **Before** hook to check that any call hitting this endpoint is having an authenticated user. If not, then we set a 401 and stop the pipeline by returning `false`.

Inside the `GET` method we can extract the user claims.

We still need to authenticate the user somewhere, in the module we only checked the authorization. We do this inside the global hooks in the `startup.cs` file:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
  if (env.IsDevelopment())
  {
    app.UseDeveloperExceptionPage();
  }

  app.UseCarter(new CarterOptions(ctx => this.GetBeforeHook(ctx)));
}

private Task<bool> GetBeforeHook(HttpContext context)
{
  var identity = new GenericIdentity("AuthenticatedUser");

  identity.AddClaim(new Claim(ClaimTypes.Email, "user@example.com"));

  context.User = new ClaimsPrincipal(identity);

  return Task.FromResult(true);
}
```

The BeforeHook is not actually checking the user, but you can do this by inspecting a cookie or preferably, checking the Json Web Token (JWT) supplied in the Authorization header.
Lets assume we have something like that, so we can just set the ClaimsPrincipal and push some claims in.





