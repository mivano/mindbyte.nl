---
published: true
title: Different ways to return a collection from a REST API
tags:
  - API
  - APIM
  - REST  
---

Returning a collection of data in a REST API can take various forms, but there are some standards and specifications that help you to take the most out of it. Lets see some examples below:

First start with no standard at all, just returning a list of the resource. Simple to use, but it lacks any information on e.g. total count or paging instructions. Clients can do little with this simple array of elements.

![https://pbs.twimg.com/media/FbfO6rKX0AExxzV.png](https://pbs.twimg.com/media/FbfO6rKX0AExxzV.png)

Next up, use `hal+json` to return an embedded collection with enough meta data so applications can traverse the next sets of data automatically. This standard also provides a nice HATEOAS solution and embedding. [https://datatracker.ietf.org/doc/html/draft-kelly-json-hal](https://datatracker.ietf.org/doc/html/draft-kelly-json-hal)

![https://pbs.twimg.com/media/FbfO7GFX0AEUF_A.jpg](https://pbs.twimg.com/media/FbfO7GFX0AEUF_A.jpg)

The `collection+json` is a richer standard to query and return data and can also return collections of data next to query for the data. For this standard, look [here](http://amundsen.com/media-types/collection/format/)

![https://pbs.twimg.com/media/FbfO7paWAAAQzDi.jpg](https://pbs.twimg.com/media/FbfO7paWAAAQzDi.jpg)

We also have `JSON+LD` which can be used to add meaning to JSON documents and as such also to collections. [https://json-ld.org/](https://json-ld.org/)

![https://pbs.twimg.com/media/FbfO8HOXkAEg3P2.jpg](https://pbs.twimg.com/media/FbfO8HOXkAEg3P2.jpg)

The `SIREN` spec is a hypermedia specification for representing entities with actions for modifying and linking. See the [details](https://github.com/kevinswiber/siren)

![https://pbs.twimg.com/media/FbfO8jMXoAIBZxC.jpg](https://pbs.twimg.com/media/FbfO8jMXoAIBZxC.jpg)

More and more populair is `JSON:API`, which is a specification for how a client should request that resources be fetched or modified, and how a server should respond to those requests. [https://jsonapi.org/](https://jsonapi.org/)

![https://pbs.twimg.com/media/FbfO9DtXoAA9ezI.jpg](https://pbs.twimg.com/media/FbfO9DtXoAA9ezI.jpg)

So what to pick? 

- Use collection+json for a full feature set 
- Pick hal+json for its simplicity but still good extensive options 
- JSON+LD to extend an existing API 
- JSON:API gets a lot of traction, so client support can be good

Most of these specifications have for support for nesting relations, HATEAOS descriptions and templating to name a few. As always; it depends on what your really need to support, so check out the options carefully.

