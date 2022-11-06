---
sitemap: false
permalink: /.well-known/webfinger
layout: null
---

{
  "subject": "acct:mivano@mastodon.social",
  "aliases": [
    "https://mastodon.social/@mivano",
    "https://mastodon.social/users/mivano"
  ],
  "links": [
    {
      "rel": "http://webfinger.net/rel/profile-page",
      "type": "text/html",
      "href": "https://mastodon.social/@mivano"
    },
    {
      "rel": "self",
      "type": "application/activity+json",
      "href": "https://mastodon.social/users/mivano"
    },
    {
      "rel": "http://ostatus.org/schema/1.0/subscribe",
      "template": "https://mastodon.social/authorize_interaction?uri={uri}"
    }
  ]
}
