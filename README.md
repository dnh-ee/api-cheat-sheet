# API Design Cheat Sheet
1. Build the API with consumers in mind--as a product in its own right.
    * Not for a specific UI.
    * Embrace flexibility / tunability of each endpoint (see #5, 8, 9, 10).
    * Eat your own dogfood, even if you have to mockup an example UI.
    * Model everything as a resource, even processes: creating an ID for a process allows for easy transition to async processing, and supports easy auditing, observability

1. Use the Collection Metaphor.
    * Two URLs (endpoints) per resource:
        * The resource collection (e.g. /orders)
        * Individual resource within the collection (e.g. /orders/{orderId}).
    * Use plural forms (‘orders’ instead of ‘order’).[^how-to-design-rest-apis]
    * Alternate resource names with IDs as URL nodes (e.g. /orders/{orderId}/items/{itemId})
    * Keep URLs as short as possible. Preferably, no more-than three nodes per URL. In particular avoid nesting of sub-resources when sub-resource can be exposed as a top-level entity in its own right (has globally unique Id of its own).[^how-to-design-rest-apis] 
    * Use `-` as the separator for path compound name components[^mark-masse-ch2]
    * Clean URLs (don't add `.json` or other extensions)[^how-to-design-rest-apis]

1. Use nouns as resource names (e.g. don’t use verbs in URLs).

1. Return decorated responses (i.e. no naked objects, arrays or maps).
    * Use `data` as the top-level data object for successful responses.[^how-to-design-rest-apis]
   
1. Make resource representations meaningful.
    * Prefer resource links over entity IDs
    * Design resource representations. Don’t simply represent database tables.
    * Merge representations. Don’t expose relationship tables as two IDs.

1. Identifier design.
   * Use strings for all identifier data.[^how-to-design-rest-apis]
   * Avoid sequential, scrapeable Ids [^build-apis-you-wont-hate]
   * Prefix identifiers by type [^how-to-design-rest-apis]

1. Use structured error format[^rcf9457]
   * Support RCF9457, particularly the `type` field with should link to unique identifier for that error 
   * For all `Logical Failures` (business errors) define error definition for each, either as unique description pages, or as tag URIs
   * For `Unexpected Errors` define small number of catch-all definitions (resource unavailable, unexpected data, panic, etc)
   * To disambiguate certain Status Code responses, e.g.:
      * 404 Valid endpoint but Not Found vs Invalid endpoint
     
1. Support filtering, sorting, and pagination on collections.

1. Support link expansion of relationships. Allow clients to expand the data contained in the response by including additional representations instead of, or in addition to, links.

1. Support field projections on resources. Allow clients to reduce the number of fields that come back in the response.

1. Use the HTTP method names to mean something:
    * `POST` - create and other non-idempotent operations. **Unsafe[^safety]. Not idempotent.**
    * `PUT` - update (replace) a known resource or collection location atomically. **Unsafe[^safety]. Idempotent.**
    * `GET` - read a resource or collection. Should not modify resources on the server. **Safe[^safety]. Idempotent.**
    * `DELETE` - remove a resource or collection. Repeating call should return `404`. **Unsafe[^safety]. Idempotent.**
    * `PATCH` - update part of a resource or collection.  **Unsafe[^safety]. Not Idempotent.**
    * `HEAD` - Same as `GET` but just return headers.

1. Use HTTP status codes to be meaningful.[^http-status-code][^rest-api-tutorial]
    * Success -
       * `200` - Request processing completed successfully. Do not use for failures.
       * `201` - Created. Returned on successful creation of a new resource. Include a 'Location' header with a link to the newly-created resource.
       * `202` - Accepted, asynchronous processing
    * Client-responsible failures -
       * `400` - Bad request. Badly structured request (does not conform to schema)
       * `422` - Invalid request data within well-structured request
       * Not Found
          * `404` - Not found. Resource may be available at a future date
          * `410` - Gone. Resource permanently deleted and will not return (**cacheable**)
       * Security Problem
          * `401` - Unauthorised. Client should authenticate and retry.
          * `403` - Forbidden. User is not permitted to access resource. Authenticating will make no difference.
       * Inconsistent client and service state -
          * `409` - Conflict. Duplicates existing data record, violates some unique key constraint, or is otherwise incompatible with existing service state.
       * Request Semantics Issues
          * `405`, `406`, `415` Method and Content negotiation problems
       * Backpressure
          * `429` - Rate limited.    
    * Service Error -
       * `500` - either a `Logical Failure` or `Unexpected Error` encountered in the service state during processing
    * Upstream Dependency Error (generally of the `Unexpected Error` category) -
       * `502` - Error received from upstream system. May be 4xx or 5xx response or 2xx with invalid data
       * `504` - Upstream dependency unavailable

1. Use ISO 8601 timepoint formats for dates in representations.
    * Don't trust platform/language defaults[^how-to-design-rest-apis]

1. Use [OAuth2](http://oauth.net/2/) to secure your API.
    * Use a Bearer token for authentication.
    * Require HTTPS / TLS / SSL to access your APIs. OAuth2 Bearer tokens demand it. Unencrypted communication over HTTP allows for simple eavesdroppping and impersonation.

1. Use Content-Type negotiation to describe incoming request payloads.

    For example, let's say you're doing ratings, including a thumbs-up/thumbs-down and five-star rating. You have one route to create a rating: **POST /ratings**

    How do you distinguish the incoming data to the service so it can determine which rating type it is: thumbs-up or five star?

    The temptation is to create one route for each rating type: **POST /ratings/five_star** and **POST /ratings/thumbs_up**

    However, by using Content-Type negotiation we can use our same **POST /ratings** route for both types. By setting the *Content-Type* header on the request to something like **Content-Type: application/vnd.company.rating.thumbsup** or **Content-Type: application/vnd.company.rating.fivestar** the server can determine how to process the incoming rating data.

1. Evolution over versioning. However, if versioning, use the Accept header instead of versioning in the URL.
    * Versioning via the Accept header is versioning the resource.
    * Expand/Contract: Additions to a JSON response do not require versioning. However, additions to a JSON request body that are 'required' are troublesome--and may require versioning.
    * Versioning via the URL signifies a 'platform' version and the entire platform must be versioned at the same time to enable the linking strategy. It should appear as the first node in the path and not version individual resources.
    * Don't pre-emptively put `/v1/` as your initial platform version. YAGNI, plus Lean Decide Late[^decide-late]

1. Consider Cache-ability. At a minimum, use the following response headers:
    * ETag - An arbitrary string for the version of a representation. Make sure to include the media type in the hash value, because that makes a different representation. (ex: ETag: "686897696a7c876b7e")
    * Date - Date and time the response was returned (in RFC1123 format). (ex: Date: Sun, 06 Nov 1994 08:49:37 GMT)
    * Cache-Control - The maximum number of seconds (max age) a response can be cached. However, if caching is not supported for the response, then no-cache is the value. (ex: Cache-Control: 360 or Cache-Control: no-cache)
    * Expires - If max age is given, contains the timestamp (in RFC1123 format) for when the response expires, which is the value of Date (e.g. now) plus max age. If caching is not supported for the response, this header is not present. (ex: Expires: Sun, 06 Nov 1994 08:49:37 GMT)
    * Pragma - When Cache-Control is 'no-cache' this header is also set to 'no-cache'. Otherwise, it is not present. (ex: Pragma: no-cache)
    * Last-Modified - The timestamp that the resource itself was modified last (in RFC1123 format). (ex: Last-Modified: Sun, 06 Nov 1994 08:49:37 GMT)

1. Provide Idempotence mechanisms
    * Ensure that your GET, PUT, and DELETE operations are all idempotent[^idempotency].  There should be no adverse side affects from operations.
    * Consider using a client generated Idempotency Key[^how-to-design-rest-apis]


[^mark-masse-ch2]: [Mark Masse REST API Design Rulebook](https://www.oreilly.com/library/view/rest-api-design/9781449317904/index.html) (Ch.2 Identifier Design with URIs)
[^http-status-code]: [Michael Kropat: Choosing an HTTP Status Code](https://www.codetinkerer.com/2015/12/04/choosing-an-http-status-code.html)
[^how-to-design-rest-apis]: [Jeff Schnitzer: How to (and how not to) design REST APIs](https://github.com/stickfigure/blog/wiki/How-to-(and-how-not-to)-design-REST-APIs)
[^rcf9457]: [IETF / M. Nottingham - Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457)
[^build-apis-you-wont-hate]: [Phil Sturgeon - Build APIs You Won't Hate](https://leanpub.com/build-apis-you-wont-hate)
[^rest-api-tutorial]: [Todd Fredrich - REST API Tutorial](https://www.restapitutorial.com/)
[^idempotency]: [Todd Fredrich - REST API Tutorial - Idempotency](http://www.restapitutorial.com/lessons/idempotency.html)
[^safety]: [Todd Fredrich - REST API Tutorial - Safety](https://www.restapitutorial.com/introduction/safety)
[^decide-late]: [Mary & Todd Poppendieck - Lean Software Development: An Agile Toolkit - Decide as Late as Possible](https://learning.oreilly.com/library/view/lean-software-development/0321150783/ch03.html)
