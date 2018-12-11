# HTTP API Design Standards

## Guidelines
This document provides guidelines and examples for GoCardless APIs, encouraging consistency, maintainability, and best practices.

### Sources:
- https://github.com/interagent/http-api-design
- https://www.gov.uk/service-manual/making-software/apis.html
- http://www.mnot.net/blog/
- http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
- https://github.com/WhiteHouse/api-standards
- http://apigee.com/about/content/api-fa%C3%A7ade-pattern
- https://pages.apigee.com/web-api-design-ebook.html
- https://groups.google.com/forum/#!forum/api-craft

## JSON API
All endpoints must follow the core [JSON API spec](http://jsonapi.org/format/)

Changes from JSON API:
- The primary resource must be keyed by its resource type. The endpoint URL must also match the resource type.
- API errors do not currently follow the JSON API spec.
- Updates should always return `200 OK` with the full resource to simplify internal logic.

Example of keying the resource by type:

```json
GET /posts/1
```

```json
{
  "posts": {
    "id": 1
  }
}
```

## JSON only
The API should only support JSON.

Reference: http://www.mnot.net/blog/2012/04/13/json_or_xml_just_decide

## General guidelines
- A URL identifies a resource.
- URLs should include nouns, not verbs.
- For consistency, only use plural nouns (e.g. "posts" instead of "post").
- Use HTTP verbs (`GET`, `POST`, `PUT`, `DELETE`) to operate on resources.
- Use filtering instead of nested resources: `/payments?subscription=xyz` rather than
  `/subscriptions/xyz/payments`. Nested resources enforce relationships that could
  change and makes clients harder to write.
- API versions should be represented as dates documented in a changelog. Version number should not be in the URL.
- API should be behind a subdomain: `api.gocardless.com`

## RESTful URLs

### Good URL examples
- List of payments:
  - `GET https://api.gocardless.com/payments`
- Filtering is a query:
  - `GET https://api.gocardless.com/payments?status=failed&sort=-created`
  - `GET https://api.gocardless.com/payments?sort=created`
- A single payment:
  - `GET https://api.gocardless.com/payments/1234`
- All amendments in (or belonging to) this subscription:
  - `GET https://api.gocardless.com/subscription_amendments?subscription=1234`
- Include nested resources in a comma separated list:
  - `GET https://api.gocardless.com/payments/1234?include=events`
- Include only selected fields in a comma separated list:
  - `GET https://api.gocardless.com/payments/1234?fields=amount`
- Get multiple resources:
  - `GET https://api.gocardless.com/payments/1234,444,555,666`
- Action on resource:
  - `POST https://api.gocardless.com/payments/1234/actions/cancel`

### Bad URL examples
- Singular nouns:
  - `GET https://api.gocardless.com/payment`
  - `GET https://api.gocardless.com/payment/123`
  - `GET https://api.gocardless.com/payment/action`
- Verbs in the URL:
  - `GET https://api.gocardless.com/payment/create`
- Nested resources:
  - `GET https://api.gocardless.com/subscriptions/1234/amendments`
- Filtering outside of query string:
  - `GET https://api.gocardless.com/payments/desc`
- Filtering to get multiple resources:
  - `GET https://api.gocardless.com/payments?id[]=11&id[]=22`

## HTTP verbs
Here's an example of how HTTP verbs map to create, read, update, delete operations in a particular context:

| HTTP METHOD   | POST           | GET          | PUT          | PATCH        | DELETE       |
|:------------- |:-------------- |:------------ |:------------ |:------------ |:------------ |
| CRUD OP       | CREATE         | READ         | UPDATE       | UPDATE       | DELETE       |
| /plans        | Create new plan| List plans   | Bulk update  | Error        | Delete all plans |
| /plans/1234   | Error          | Show Plan If exists | If exists, full/partial update Plan; If not, error | If exists, update Plan using JSON Patch format; If not, error | Delete Plan |

## Actions

Avoid resource actions. Create separate resources where possible.

#### Good
```http
POST /refunds?payment=ID&amount=1000
```

#### Bad
```http
POST /payments/ID/refund
```

Where special actions are required, place them under an `actions` prefix.
Actions should always be idempotent.

```http
POST /payments/ID/actions/cancel
```

## Responses
Don’t set values in keys.

#### Good

```json
"tags": [
  {"id": "125", "name": "Environment"},
  {"id": "834", "name": "Water Quality"}
]
```

#### Bad

```json
"tags": [
  {"125": "Environment"},
  {"834": "Water Quality"}
]
```

## String IDs
Always return string ids. Some languages, like JavaScript, don't support big ints. Serialize/deserialize ints to strings if storing ids as ints.

## Error handling
Error responses should include a message for the user, an internal error type (corresponding to some
specific internally determined constant represented as a string), and links to info for developers.

There must only be one top level error. Errors should be returned in turn. This makes internal error
logic and dealing with errors as a consumer of the API easier.

Validation and resource errors are nested in the top level error under `errors`.

The error is nested in `error` to make it possible to add, for example, deprecation errors on successful requests.

The HTTP status `code` is used as a top level error, `type` is used as a sub error, and nested
`errors` may have more specific `type` errors, such as `invalid_field`.

### Formating errors vs integration errors
Formatting errors should be separate from errors handled by the integration.

Formatting errors include things like field presence and length, and are returned when incorrect
data is sent to the API.

An example of an error that should be handled by the integration is attempting to create a payment
against a mandate that has expired. This is an edge case that the API integration needs to
handle. Do not mask these errors as validation errors. Return them as a top level error instead.

### Top level error

- Top level errors MUST implement `request_id`, `type`, `code` and `message`.
- `type` MUST relate to the `reason`. For example, use it to categorise the error: `reason: api_error`.
- `message` MUST be specific.
- Top level errors MAY implement `documentation_url`, `request_url` and `id`.
- Only return `id` for server errors (5xx). The `id` should point to the exception you track internally.

```json
{
  "error": {
    "documentation_url": "https://api.gocardless.com/docs/beta/errors#access_forbidden",
    "request_url": "https://api.gocardless.com/requests/REQUEST_ID",
    "request_id": "REQUEST_ID",
    "id": "ERROR_ID",
    "type": "access_forbidden",
    "code": 403,
    "message": "You don't have the right permissions to access this resource"
  }
}
```

### Nested errors

- Nested errors MUST implement `reason` and `message`.
- `reason` MUST be specific to the error.
- Nested errors MAY implement `field`.

```json
{
  "error": {
    "top level errors": "...",

    "errors": [{
      "field": "account_number",
      "reason": "missing_field",
      "message": "Account number is required"
    }]
  }
}
```

### HTTP status code summary

- `200 OK` - everything worked as expected.
- `400 Bad Request` - e.g. invalid JSON.
- `401 Unauthorized` - no valid API key provided.
- `402 Request Failed` - parameters were valid but request failed.
- `403 Forbidden` - missing or invalid permissions.
- `404 Not Found` - the requested item doesn’t exist.
- `422 Unprocessable Entity` - parameters were invalid/validation failed.
- `500, 502, 503, 504 Server errors` - something went wrong on GoCardless’ end.

#### 400 Bad Request

- When the request body contains malformed JSON.
- When the JSON is valid but the document structure is invalid (e.g. passing an array when an object should be passed).

#### 422 Unprocessable Entity

- When model validations fail for fields (e.g. name too long).
- Trying to create a resource when a related resource is in a bad state.

## What changes are considered “backwards-compatible”?

- Adding new API resources.
- Adding new optional request parameters to existing API methods.
- Adding new properties to existing API responses.
- Changing the order of properties in existing API responses.
- Changing the length or format of object IDs or other opaque strings.
  - This includes adding or removing fixed prefixes (such as ch_ on charge IDs).
  - You can safely assume object IDs we generate will never exceed 128 characters, but you should be
    able to handle IDs of up to that length. If for example you’re using MySQL, you should store IDs
    in a VARCHAR(128) COLLATE utf8_bin column (the COLLATE configuration ensures case-sensitivity in
    lookups).
- Adding new event types. Your webhook listener should gracefully handle unfamiliar events types.

## Versioning changes

The versioning scheme is designed to promote incremental improvement to the API and discourage rewrites.

Server initiated events such as webhooks should not contain serialised resources. If a resource
changed, provide its id instead and let the client request it using a version.

### Format

Versions should be dated as ISO8601 (YYYY-MM-DD)
- Good: 2014-05-04
- Bad: v-1.1, v1.2, 1.3, v1, v2

### Version maintenance

Maintain old API versions for at least 6 months.

### Implementation guidelines

The API version must be set using a custom HTTP header. The API version must not be defined in the
URL structure (e.g. `/v1`) because it makes incremental change impossible.

#### HTTP Header

`GoCardless-Version: 2014-05-04`

Enforce the header on all requests.

Validate the version against available versions. Do not allow dates up to a version.

The API changelog must only contain backwards-incompatible changes. All non-breaking changes are automatically available to old versions.

Reference: https://stripe.com/docs/upgrades

## X-Headers

The use of `X-Custom-Header` has [been deprecated](http://tools.ietf.org/html/rfc6648).

## Resource filtering
Resource filters MUST be in singular form.

Multiple ids should be supplied to a filter as a comma separated list, and should be translated into an `OR` query. Chaining multiple filters with `&` should be translated into an `AND` query.

#### Good

```http
GET /refunds?payment=ID1,ID2&customer=ID1
```

#### Bad

```http
GET /refunds?payments=ID1,ID2&customer=ID1
```

## Pagination

All list/index endpoints must be paginated by default. Pagination must be reverse chronological.

Only support cursor or time based pagination.

#### Defaults

`limit=50`
`after=NEWEST_RESOURCE`
`before=null`

#### Limits

`limit=500`

Parameters:

| Name        | Type           | Description  |
| ------------- |:-------------:| -----:|
| `after`     | string | id to start after |
| `before`      | string      |   id to start before |
| `limit` | string      |   number of records |

### Response

Paginated results are always enveloped:

```
{
  "meta": {
    "cursors": {
      "after": "abcd1234",
      "before": "wxyz0987"
    },
    "limit": 50
  },
  "payments": [{
    …
  },
  …]
}
```

## Updates
Full or partial updates using `PUT` should replace any parameters passed and ignore fields not submitted.

```
GET /items/id_123
{
  "id": "id_123",
  "meta": {
    "created": "date",
    "published": false
  }
}
```

```
PUT /items/id_123 { "meta": { "published": true } }
{
  "id": "id_123",
  "meta": {
    "published": false
  }
}
```

### PATCH Updates

PATCH is reserved for [JSON Patch](http://jsonapi.org/format/#patch) operations.

## JSON encode POST, PUT & PATCH bodies

`POST`, `PUT` and `PATCH` expect JSON bodies in the request. `Content-Type` header MUST be set to `application/json`.
For unsupported media types a `415` (Unsupported Media Type) response code is returned.

## Caching

Most responses return an `ETag` header. Many responses also return a `Last-Modified` header. The
values of these headers can be used to make subsequent requests to those resources using the
`If-None-Match` and `If-Modified-Since` headers, respectively. If the resource has not changed, the
server will return a `304 Not Modified`. Note that making a conditional request and receiving a 304
response does _not_ count against your rate limit, so we encourage you to use it whenever possible.

`Cache-Control: private, max-age=60`
`ETag: <hash of contents>`
`Last-Modified: updated_at`

#### Vary header
The following header values must be declared in the Vary header: `Accept`, `Authorization` and `Cookie`.

Any of these headers can change the representation of the data and should invalidate a cached
version. This can be useful if users have different accounts to do admin, each with different
privileges and resource visibility.

Reference: https://www.mnot.net/cache_docs/

## Compression
All responses should support gzip.

## Result filtering, sorting & searching
See JSON-API: http://jsonapi.org/format/#fetching-filtering

## Pretty printed responses
JSON responses should be pretty printed.

## Time zone/dates
Explicitly provide an ISO8601 timestamp with timezone information (DateTime in UTC).
Use the exact timestamp for API calls that allow a timestamp to be specified.
These timestamps look something like `2014-02-27T15:05:06+01:00`. ISO 8601 UTC format: YYYY-MM-DDTHH:MM:SSZ.

## HTTP rate limiting
All endpoints must be rate limited. The current rate limit status is returned in the HTTP headers of
all API requests.

```http
Rate-Limit-Limit: 5000
Rate-Limit-Remaining: 4994
Rate-Limit-Reset: Thu, 01 Dec 1994 16:00:00 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Retry-After: Thu, 01 May 2014 16:00:00 GMT

RateLimit-Reset uses the HTTP header date format: RFC 1123 (Thu, 01 Dec 1994 16:00:00 GMT)
```

Exceeding rate limit:
```http
// 429 Too Many Requests
{
    "message": "API rate limit exceeded.",
    "type": "rate_limit_exceeded",
    "documentation_url": "http://developer.gocardless.com/#rate_limit_exceeded"
}
```

## CORS
Support Cross Origin Resource Sharing (CORS) for AJAX requests.

Resources:
- [CORS W3C working draft](https://www.w3.org/TR/cors/)
- [HTML5 Rocks](http://www.html5rocks.com/en/tutorials/cors/)

Any domain that is registered against the requesting account is accepted.

```http
$ curl -i https://api.gocardless.com -H "Origin: http://dvla.com"
HTTP/1.1 302 Found
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: ETag, Link, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, OAuth-Scopes, Accepted-OAuth-Scopes
Access-Control-Allow-Credentials: false

// CORS Preflight request
// OPTIONS 200
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-Modified-Since, If-None-Match, If-Unmodified-Since, Requested-With
Access-Control-Allow-Methods: GET, POST, PATCH, PUT, DELETE
Access-Control-Expose-Headers: ETag, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: false
```

## TLS/SSL
All API request MUST be made over SSL, including outgoing web hooks. Any non-secure requests
return `ssl_required`, and no redirects are performed.

```http
HTTP/1.1 403 Forbidden
Content-Length: 35

{
  "message": "API requests must be made over HTTPS",
  "type": "ssl_required",
  "docs": "https://developer.gocardless.com/errors#ssl_required"
}
```

## Include related resource representations
See JSON-API: http://jsonapi.org/format/#fetching-includes

## Limit fields in response
See JSON-API: http://jsonapi.org/format/#fetching-sparse-fieldsets

## Unique request identifiers
Set a `Request-Id` header to aid debugging across services.
