# SquareBoat API Spec Document

The aim of this document is to highlight what we believe are good practices for RESTful API design. We do understand that there are no right answers here, but some approaches have fewer cons than others.

## Caching

###ETag:

Include an ETag header in all responses, identifying the specific version of the returned resource.

Once the client receives the response and sees the ETag value, it should save the ETag and the response in the local store and issue subsequent requests to the same data with the If-None-Match header that points to the value of the ETag saved:

```
Get /countries HTTP/1.1
Host: api.example.org
Content-Type: application/vnd.uber+json
If-None-Match: "88d979a0a78942b5bda05ace4214556a"
```

If the data-set hasn’t modified on the server, the server must respond with HTTP 304 and an empty body:

```
HTTP/1.1 304 Not Modified
Content-Type: application/vnd.uber+json
Expires: Sat, 01 Jan 1970 00:00:00 GMT
Pragma: no-cache
Content-Length: 0
```

If the data-set has modified on the server, the server must respond with a full HTTP 200 response and the new value of ETag.

###Last-Modified:

In this workflow, for each response, the server provides a “Last-Modified” header in the response, containing the last date the specific data was modified:

```
HTTP/1.1 200 OK
Last-Modified: Mon, 7 Dec 2015 15:29:14 GMT
Content-Length: 23456
Content-Type: application/vnd.uber+json; charset=UTF-8
… the rest of the response …
```

Once the client receives the response and sees the Last-Modified header, it should save the value of the Last-Modified date-time and the corresponding response in the local store (cache). The client should issue subsequent requests to the same data with the If-Modified-Since header that points to the value of the date-time saved:

```
Get /countries HTTP/1.1
Host: api.example.org
Content-Type: application/vnd.uber+json
If-Modified-Since: Mon, 7 Dec 2015 15:29:14 GMT
```

If the data-set hasn’t modified on the server since the date-time indicated, the server must respond with HTTP 304 and an empty body:

HTTP/1.1 304 Not Modified
Content-Type: application/vnd.uber+json
Content-Length: 0

If the data-set has modified on the server, the server must respond with a full HTTP 200 response and the new value of the “Last-Modified” header.

## Rate limiting
To prevent abuse, it is now standard practice to add some sort of rate limiting to an API. HTTP status code `429 Too Many Requests` will be returned if you exceed the prescribed rate limit.

## JSON only responses
It's time to leave XML behind in APIs. It's verbose, it's hard to parse, it's hard to read, its data model isn't compatible with how most programming languages model data and its extendibility advantages are irrelevant when your output representation's primary needs are serialization from an internal representation.

## SSL always
One thing to watch out for is non-SSL access to API URLs. We do not redirect these to their SSL counterparts and throw a hard error instead! The last thing you want is for poorly configured clients to send requests to an unencrypted endpoint, just to be silently redirected to the actual encrypted endpoint.

## Headers:
We enable Cross-Origin Resource Sharing (CORS) by default.

## Keep JSON minified in all responses
Extra whitespace adds needless response size to requests, and many clients for human consumption will automatically "prettify" JSON output. It is best to keep JSON responses minified.

## HTTP methods

Best explained by examples:

```
GET /tickets - Retrieves a list of tickets
GET /tickets/12 - Retrieves a specific ticket
POST /tickets - Creates a new ticket
PUT /tickets/12 - Updates ticket #12
PATCH /tickets/12 - Partially updates ticket #12
DELETE /tickets/12 - Deletes ticket #12
```

Use nouns but no verbs. Keep it simple and use only plural nouns for all resources.

```
/cars - Returns a list of cars
/cars/71 - Returns a specific car
```

Do not use verbs:

```
/getAllCars
/createNewCar
/deleteAllRedCars
```

If a resource is related to another resource, use subresources whenever possible.

```
GET /cars/711/drivers/ Returns a list of drivers for car 711
GET /cars/711/drivers/4 Returns driver #4 for car 711
```

## JSON Data vs Form Data:

We use JSON over form data because complicated nested forms are represented more easily in JSON than form data.

## Filtering

GET /cars?color=red&sort=desc
GET /cars?fields=manufacturer,model,id,color

## Versioning:

Our preferred URL structure would be `/blog/api/v1`

Avoid v1.1 etc.

## Response Codes:

| Code    | Alias                   | Description |
| ------- | :----------------------:| ----------- |
| **200** | *OK*                    | Everything is working as expected.     |
| **201** | *Created*               | Response to a POST that results in a creation. Should be combined with a Location header pointing to the location of the new resource  |
| **204** | *No Content*            | Response to a successful request that won't be returning a body (like a DELETE request) |
| **304** | *Not Modified*          | The client can use cached data |
| **400** | *Bad Request*           | The request was invalid or cannot be served. The exact error will be explained in the error payload. This occurs often due to missing a required parameter. |
| **401** | *Unauthorized*          | The request requires an user authentication. |
| **403** | *Forbidden*             | The server understood the request, but is refusing it or the access is not allowed / When authentication succeeded but authenticated user doesn't have access to the resource. |
| **404** | *Not Found*             | There is no resource behind the URI. |
| **405** | Method not Allowed      | When an HTTP method is being requested that isn't allowed for the authenticated user. |
| **408** | *Request Timed Out*     | The server timed out while waiting for the request. |
| **410** | *Gone*                  |  Indicates that the resource at this end point is no longer available. Useful as a blanket response for old API versions. |
| **422** | *Unprocessable Entity*  | Used for validation errors. Should be used if the server cannot process the enitity, e.g. if an image cannot be formatted or mandatory fields are missing in the payload. |
| **429** | *Too Many Requests*     | Too Many Requests - When a request is rejected due to rate limiting. |
| **500** | *Internal Server Error* | API developers should avoid this error. If an error occurs in the global catch blog, the stracktrace should be logged and not returned as response.|

## Error Responses:

Some folks will try to use HTTP status codes exclusively and skip using error codes because they do not like the idea of making their own error codes or having to document them, but this is not a scalable approach.

There will be some situations where the same endpoint could easily return the same status code for more than one different condition. The status codes are there to merely hint at the category of error relying on the actual error code and error message provided in the HTTP response to include more information in case the client is interested.

```
{
  "code" : 1234,
  "message" : "Something bad happened :(",
  "description" : "More details about the error here"
}

{
  "code" : 1024,
  "message" : "Validation Failed",
  "errors" : [
    {
      "code" : 5432,
      "field" : "first_name",
      "message" : "First name cannot have fancy characters"
    },
    {
       "code" : 5622,
       "field" : "password",
       "message" : "Password cannot be blank"
    }
  ]
}
```

- No internal names will be exposed in the API (for example, “node” and “taxonomy term”)

## Pagination:

We will allow the client to request the number of items it would like returned per HTTP request.

```
GET /magazines?limit=25&offset=50
```

If no limit is specified, return results with a default limit.

```
{
  "metadata": {
    "resultset": {
      "count": 227,
      "offset": 25,
      "limit": 25
    }
  },
  "results": [...]
}
```

## Successful Responses

```
{
  "data": [{
	"name": "Hulk Hogan",
	"id": "100002"
  },
  {
    "name": "Mick Foley",
    "id": "100003"
  }
}
```

We would Nest foreign key relations like this (Correct way):

```
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  // ...
}
```

Instead of this (Wrong way):

```
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  // ...
}
```

- A PUT, POST or PATCH call may make modifications to fields of the underlying resource that weren't part of the provided parameters (for example: created_at or updated_at timestamps). To prevent an API consumer from having to hit the API again for an updated representation, have the API return the updated (or created) representation as part of the response.

## Naming Conventions

Since we are using JSON as your primary representation format, the "right" thing to do is to follow JavaScript naming conventions - and that means camelCase for field names


## Embedding related resources

It can be hard to pick between subresource URLs or embedded data. Embedded data can be rather difficult to pull off

GET /tickets/12?embed=customer.name,assigned_user
```
{
  "id" : 12,
  "subject" : "I have a question!",
  "summary" : "Hi, ....",
  "customer" : {
    "name" : "Bob"
  },
  assigned_user: {
   "id" : 42,
   "name" : "Jim",
  }
}
```

## No auto increment ID’s

- We do not want people to know how many resources we have
- We do not want developers to automate a scraping script by going to /resource/1, /resource/2 etc. and scraping everything away

## Enable GZip

Gzip will be enabled by default in production on all API responses.

## Transforming API output

- You want true or false as an actual JSON boolean, not a numeric string or a char(1).

- Never directly expose db fields because if you add new fields later, the api structure will also change.

## Documentation Tools (TODO)

```
http://apidocjs.com/
http://swagger.io/
http://raml.org/index.html
https://github.com/lord/slate
https://github.com/Thibaut/devdocs
http://www.programmableweb.com/news/web-api-documentation-best-practices/2010/08/12
http://bradfults.com/the-best-api-documentation/
```

Some good examples of API documentation:

```
https://dev.twitter.com/overview/documentation
https://stripe.com/docs/api#charges
https://www.twilio.com/docs/api/rest
```

### Credits / References

http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
