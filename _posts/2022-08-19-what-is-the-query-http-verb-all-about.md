---
published: true
title: What is the QUERY HTTP verb all about
tags:
  - API
  - REST
---

Did you ever hear about the QUERY HTTP verb in your REST API? You might know the GET and POST, but when to use the newly proposed QUERY method? First; why are GET or POST not enough...

While using a GET to query for data, you pass the parameters as part of the URI. Comes with risks: you might reach a size limit and need to encode. Selecting return fields is difficult. So what about a POST instead?

![https://pbs.twimg.com/media/Fagk5jiXEAI8YOw.png](https://pbs.twimg.com/media/Fagk5jiXEAI8YOw.png)

Using the POST, you pass the query in the body. However, this is not an idempotent operation. QUERY to the rescue...

![https://pbs.twimg.com/media/Fagk56gWAAATWM4.png](https://pbs.twimg.com/media/Fagk56gWAAATWM4.png)

With a QUERY you send the parameters as part of the body, but unlike POST, it is safe and idempotent, so ready for caching and retriable. While a GET returns a representation of the resource, QUERY can return a different type. So what does it return?

When nothing is found, return 204 (No content) 
Optionally a 3xx (redirect) to a location with the results or the actual results as requested.

![https://pbs.twimg.com/media/Fagk6ZDWAAA8Zof.jpg](https://pbs.twimg.com/media/Fagk6ZDWAAA8Zof.jpg)

So a safe, idempotent, cachable HTTP method . Supporting accept headers and REST semantics. It add clarity and intent to an API. 

The drawback? It is still a draft so support is limited. Read more [https://datatracker.ietf.org/doc/draft-ietf-httpbis-safe-method-w-body/](https://datatracker.ietf.org/doc/draft-ietf-httpbis-safe-method-w-body/)

So are you planning to use the QUERY soon? 