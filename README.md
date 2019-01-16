## REST-API Cheat Sheet [see also](https://hersengarage.nl/rest-api-design-as-a-craft-not-an-art-a3fd97ed3ef4)

Initially created for Dutch Railways NS. Based on [this cheatsheet](https://github.com/RestCheatSheet/api-cheat-sheet).

- Build the API with consumers (developers) in mind--as a product in its own right.

  * Not for a specific front-end.
  * Use a domain model ([example domain model](https://imgur.com/55qxMz6h.png)).
  * Use use-cases and scenarios to validate your APIs UX.

- A good API (or any other interface) is

  * Consistent (avoid surprises) 
  * Cohesive (only lists purposeful endpoints)
  * Complete (has all necessary endpoints for its purpose, but no more)
  * Minimal (only one way to do things)
  * Encapsulating (hiding implementation details)
  * Self explaining
  * Documented

- Base resources and URLs on the entities and relationships of your domain model.

- Create a Swagger or OpenAPI file for your API before you start implementing the REST-server

- Use a naming convention

  * Use plural forms for resources (```orders``` instead of ```order```), it's the datamodelling standard.
  * Use lowercase in constant parts of paths, e.g: ```/lowercase```, not ```/CamelCase``` or ```/UPPERCASE```.
  * Use camelCase field names, e.g.: ```fieldName```, not ```FieldName``` or ```field_name```
  * Use CamelCase object names, e.g.: ```TicketObject```, not ```ticketObject``` or ```ticket_object```

- Use the Collection Metaphor.

    * Two URLs per public resource in the domain model:

      * The resource collection (e.g. ```/orders```)
      * Individual resource within the collection (e.g. ```/orders/{orderId}```).

            e.g.

            POST /orders
            GET /orders
            GET /orders/{orderId}
            PUT /orders/{orderId}
            PATCH /orders/{orderId}
            DELETE /orders/{orderId}

- Reflect the hierarchy of the domain model in the URLs

  * Parts are always in the path to their wholes (the composition or 1..n relationship).
  * Alternate resource names with IDs as URL parts.

        e.g.

        GET /wholes/{wholeId}
        GET /wholes/{wholeId}/parts/{partId}
        GET /wholes/{wholeId}/parts/{partId}/subparts/{subpartId}


- Evolution over versioning.

  * Versioning via the URL signifies a 'platform' version and the entire platform must be versioned at the same time to enable the linking strategy.

        e.g.

        https://api.example.com/1/orders

  * Versioning via the Accept header is versioning the resource.

        e.g.

        GET /jobs HTTP/1.1
        Host: api.example.com
        Accept: application/vnd.example.api+json;version=2

  * Additions to a response do not require versioning. However, additions to a request body that are 'required' are troublesome--and may require versioning (breaking changes).
  * Hypermedia linking (HATEOAS) and versioning is troublesome no matter what--minimize it.

- Use the HTTP verbs to mean this:

    * POST - create and other non-idempotent operations.
    * PUT - replace.
    * PATCH - (partial) update.
    * GET - read a resource or collection.
    * DELETE - remove a resource or collection.

- Ensure that your GET, PUT, PATCH and DELETE operations are all [idempotent](http://www.restapitutorial.com/lessons/idempotency.html).

- Use [HTTP status codes](https://httpstatuses.com/) to be meaningful.
  * 200 - Success.
  * 201 - Created. Returned on successful creation of a new resource. Include a 'Location' header with a link to the newly-created resource.
  * 203 - Accepted.
  * 204 - No content (for empty responses)
  * 400 - Bad request. Data issues such as invalid JSON, etc.
  * 404 - Not found. Resource not found on GET.
  * 409 - Conflict. Duplicate data or invalid data state would occur.

- Make resource representations meaningful.

  * No plain IDs embedded in responses. Use links and reference objects.
  * Design resource representations not database entities (use a (logical) domain model not a (technical) data model).
  * Merge representations. Don’t expose internal / coupling tables as two IDs.

- Support sorting and pagination on collections (```?offset=100&limit=50&order=id```).

- Support link expansion of relationships. Allow clients to expand the data contained in the response by including additional representations instead of, or in addition to, links.

- Allow clients to select the fields that come back in the response (with query-arguments, ```?fields=name&fields=address&fields=city```)

- Use ISO 8601 for dates. 

      E.g. 

      1997-07-16T19:20:30+01:00   # time-offset
      1997-07-16T19:20:30Z        # Zulu-time
      1997-07-16T19:20:30EST      # time-zone
      1997-07-16T19:20:30         # local datetime

- Use ISO 4217 for currency codes.

- Use ISO 3166 for country codes.

- Use [OAuth2](http://oauth.net/2/) to secure your API.
  * Use an auto-expiring Bearer token for authentication (```Authorisation: Bearer f0ca4227-64c4-44e1-89e6-b27c62ac2eb6```).
  * Require HTTPS.

- Enforce use of the Content-Type and Accept-Type headers even if you use JSON as default for both requests and responses.

      e.g.

      Content-Type: application/json
      Accept-Type: application/json

- Enforce the use the Accept-Language header for locale setting (```Accept-Language: nl, en-gb;q=0.8, en;q=0.7```

- Consider Cache-ability. At a minimum, use the following response headers:
    * ETag - An arbitrary string for the version of a representation. Make sure to include the media type in the hash value, because that makes a different representation. (```ETag: "686897696a7c876b7e"```)
    * Date - Date and time the response was returned (in RFC1123 format). (```Date: Sun, 06 Nov 1994 08:49:37 GMT```)
    * Cache-Control - The maximum number of seconds (max age) a response can be cached. However, if caching is not supported for the response, then no-cache is the value. (```Cache-Control: 360``` or ```Cache-Control: no-cache```)
    * Expires - If max age is given, contains the timestamp (in RFC1123 format) for when the response expires, which is the value of Date (e.g. now) plus max age. If caching is not supported for the response, this header is not present. (```Expires: Sun, 06 Nov 1994 08:49:37 GMT```)
    * Pragma - When Cache-Control is 'no-cache' this header is also set to 'no-cache'. Otherwise, it is not present. (```Pragma: no-cache```)
    * Last-Modified - The timestamp that the resource itself was modified last (in RFC1123 format). (```Last-Modified: Sun, 06 Nov 1994 08:49:37 GMT```)


