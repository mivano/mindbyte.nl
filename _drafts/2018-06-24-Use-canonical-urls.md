---
published: false
tags:
  - api
  - http
category:
  - HTTP APIs
categories:
  - HTTP APIs
title: Canonical URLs in HTTP API
---
When you are building a HTTP api, you need to make sure your endpoints have a canonical uri structure. You can describe a canonical url as your preferred endpoint. When you have endpoints that reference the same resource, it is best to make sure there is one preferred version of this resource.

Take for example `blog` with `posts`. You can model this like:

```
GET /Blogs/{id}/Posts
```

to retrieve all the Posts for the given blog. Although this might sound simple, how do we represent a single `post`?

Would you use:

```
GET /Blogs/{blogId}/Posts/{id}
```

or 

```
GET /Posts/{id}
```

You might even have both endpoints available, but in the end they do point to the same resource, or better, the same post. So the URI for the same post is no longer unique and the client/browser might take two cache entries for the same item as they are known by different URIs.

Another example; what if you have a `users` endpoint and you want to be able to get a `user` by its identifier or by its username. 

```
GET /Users/{id:guid}
GET /Users/{name}
```

When the user is found, you will get the same record back regardless of the endpoint choosen. 

Although it is fine to have multiple endpoints for the same resource, there should ideally only be one endpoint that returns the representation of the resource. See also https://tools.ietf.org/html/rfc6596. 

This helps with cache invalidation although it might introduce more roundtrips.

In order to tell the browser that the actual resouce can be found at another locatio, you return a 3xx code.  For example a [302 (Found)](https://httpstatuses.com/302) or a [303 (See Other)](https://httpstatuses.com/303). Inside this response you include a `Location` header with the new [absolute](https://tools.ietf.org/html/rfc2616#section-14.30) URI of the resource. 

In ASP.NET (core) you might end up with something like this:

```csharp
    [Route("api/[controller]")]
    [ApiController]
    public class UsersController : ControllerBase
    {
        private readonly IUserRepository _userRepository;

        public UsersController(IUserRepository userRepository)
        {
            _userRepository = userRepository;
        }            

        // GET api/users/slug
        [HttpGet("{slug}")]
        public ActionResult<UserResource> GetByName(string slug)
        {
            var user = _userRepository.FindBySlug(slug);

            if (user == null)
                return NotFound();

            return RedirectToAction(nameof(GetById), new {id = user.Id});
        }

        // GET api/users/guid
        [HttpGet("{id:guid}")]
        public ActionResult<UserResource> GetById(Guid id)
        {
            var user = _userRepository.GetById(id);

            if (user == null)
                return NotFound();

            return Ok(user);
        }

    }
```

A call to the user by its name returns a redirect to the preferred endpoint with the identifier. If a client already has this item, it can retrieve it from its cache.

