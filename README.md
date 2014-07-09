# HTTP API Design Standards

## Guidelines
This document provides guidelines and examples for GoCardless APIs, encouraging consistency, maintainability, and best practices.

Sources:
https://github.com/interagent/http-api-design
https://www.gov.uk/service-manual/making-software/apis.html
http://www.mnot.net/blog/
http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
https://github.com/WhiteHouse/api-standards
http://apigee.com/about/content/api-fa%C3%A7ade-pattern
https://pages.apigee.com/web-api-design-ebook.html
https://groups.google.com/forum/#!forum/api-craft

## JSON API
All endpoints must follow the core JSON API spec: http://jsonapi.org/format/

Changes:
- The primary resource must be keyed by ther resource type. The endpoint url must also match the resource type.
- API errors currently do not follow the JSON API spec.

Example:

```
GET /posts/1
```

```json
{
  "posts": {
    "id": 1
  }
}
```

## JSON Schema
Provide a JSON Schema that describes what resources are available via the API, what their URLs are, how they are represented and what operations they support.
Example intended uses for the schema include:
- Auto-creating client libraries for your favorite programming language
- Generating up-to-date reference docs
- Writing automatic acceptance and integration tests

`GET https://api.gocardless.com/schema`

Sources:
http://json-schema.org/examples.html
http://tools.ietf.org/html/draft-zyp-json-schema-04

## JSON only
The API should only support JSON. Provide API clients to work with the response.

Sources:
http://www.mnot.net/blog/2012/04/13/json_or_xml_just_decide

## RESTful URLs
### General guidelines for RESTful URLs
- A URL identifies a resource.
- URLs should include nouns, not verbs.
- Use plural nouns only for consistency (no singular nouns).
- Use HTTP verbs (GET, POST, PUT, DELETE) to operate on the collections and elements.
- Never nest resources - it enforces relationships that could change, and makes clients harder to write. More info available here.
- E.g. use `/payments?subscription_id=xyz` rather than `/subscriptions/xyz/payments`
- Formats should be in the form of `/resource/{id}`
- Versions should be represented as dates documented in a changelog. Version number should not be in the url. Use Accept header for version:
- `Accept: application/vnd.gocardless.com+json; version=YYYYMMDD`
- `Vary: Accept`
- URL v. header:
- If it changes the resource returned, put it in the URL.
- If it only changes the representation of the resource returned, like OAuth info, Accept headers and Language put it in the header.
- API should be behind a subdomain: `api.gocardless.com`

### Good URL examples
- List of payments:
  - GET https://api.gocardless.com/payments
- Filtering is a query:
  - GET https://api.gocardless.com/payments?year=2011&sort=desc
  - GET https://api.gocardless.com/payments?topic=economy&year=2011
- A single payment:
  - GET https://api.gocardless.com/payments/1234
- All amendments in (or belonging to) this subscription:
  - GET https://api.gocardless.com/subscription_amendments?subscription_id=1234
- Include nested resources in a comma separated list:
  - GET https://api.gocardless.com/payments/1234?include=events
- Include only selected fields in a comma separated list:
  - GET https://api.gocardless.com/payments/1234?fields=amount
- Action on resource:
  - POST https://api.gocardless.com/payments/1234/actions/cancel

### Bad URL examples
- Non-plural noun:
  - https://api.gocardless.com/v1/payment
  - https://api.gocardless.com/v1/payment/123
  - https://api.gocardless.com/v1/payment/action
- Verb in URL:
  - https://api.gocardless.com/v1/payment/create
- Nested resources:
  - https://api.gocardless.com/v1/subscriptions/1234/amendments
- Filter outside of query string
  - https://api.gocardless.com/v1/payments/desc

## HTTP Verbs
HTTP verbs, or methods, should be used in compliance with their definitions under the HTTP/1.1 standard. The action taken on the representation will be contextual to the media type being worked on and its current state. Here's an example of how HTTP verbs map to create, read, update, delete operations in a particular context:

### TABLE
HTTP METHOD POST  GET PUT PATCH DELETE
CRUD OP CREATE  READ  UPDATE  UPDATE  DELETE
/plans  Create new plan List plans  Bulk update Error Delete all plans
/plans/1234 Error Show Plan If exists, replace Plan; If not, error  If exists, update Plan; If not, error Delete Plan

## Actions
Avoid resource actions. Create separate resources where possible:
- Bad:
```
POST /payments/ID/refund
```
- Good:
```
POST /refunds?payment=ID&amount=1000
```

Where special actions are required, place them them under a `actions` prefix:
```
POST /payments/ID/actions/cancel
```

Actions should always be idempotent.

## Responses
No values in keys

### Good:
No values in keys:
```
"tags": [
  {"id": "125", "name": "Environment"},
  {"id": "834", "name": "Water Quality"}
]
```

### Bad:
Values in keys:
```
"tags": [
  {"125": "Environment"},
  {"834": "Water Quality"}
]
```

## String IDs
Always return string ids, some languages like JavaScript don't support big ints. Serialize/deserialize ints to strings if storing ids as ints.

## Error handling
Error responses should include a message for the user, internal error type (corresponding to some specific internally determined constant, which must be represented as a string), links where developers can find more info.

There must only be one top level error. If many things are wrong, these should be returned in turn. This makes internal error logic simpler. It also makes it easier for API consumers to deal with errors.

Validation and resource errors are nested in the top level error under `errors`.
Bulk errors follow the same format as validation errors with an added fields indicating which request object caused the error.

The error is nested in `error` to make it possible to add, for example, deprecation errors on successful requests.

The HTTP status `code` is used as a top level error, `type` is used a sub error. Nested `errors` may have more specific `type` errors like `invalid_field`.

## Top level error
Top level errors MUST implement `request_id`, `type`, `code`,`message`.
`type` MUST be specific to the error.
`message` MUST be specific.
Top level errors MAY implement `documentation_url`, `request_url `, `id `.

```json
{
  "error": {
    "documentation_url": "https://api.gocardless.com/docs/beta/errors#access_forbidden",
    "request_url": "https://api.gocardless.com/requests/REQUEST_ID",
    "request_id": "REQUES_ID",
    "id": "ERROR_ID",
    "type": "access_forbidden",
    "code": 403,
    "message": "You don't have the right permissions to access this resource"
  }
}
```

## Nested errors
Nested errors MUST implement `type`,`message`.
`type` MUST be specific to the error.
Nested errors MAY implement `field`.

```json
{
  "error": {
    "top level errors": "...",

    "errors": [{
      "field": "account_number",
      "type": "missing_field",
      "message": "Account number is required"
    }]
  }
}
```

### Error types

## TABLE
Error Type  Error Description
not_found This means a resource does not exist.
deprecated  Resource has been deprecated. Please see API release notes
validation_failed Resource validation failed.
validation_failed[missing_field]  This means a required field on a resource has not been set.
validation_failed[invalid_field]  This means the formatting of a field is invalid. The documentation for that resource should be able to give you more specific information.
validation_failed[already_exists] An existing resource already has this associated resource.
validation_failed[deprecated_field] Field has been deprecated in, please see API release notes.
validation_failed[not_found]  Related resource was not found
amount_limit_exceeded Mandate amount limit exceeded.
cancellation_failed A cancellation is not possible due to the current payment state.
retry_failed  A retry is not possible due to the current payment state.
refund_failed A refund is not possible due to the current payment state.
chargeback_failed A chargeback is not possible due to the current payment state.
delete_failed The resource can't be hidden/deleted because of a constraint (e.g. default accounts can't be deleted)
api_maintenance API down for scheduled maintenance
invalid_user_credentials  For temporary access tokens only.
rate_limit_exceeded Rate limit
temporary_access_token_expired  When using temporary access tokens
api_temporarily_unavailable API is down
invalid_format  Request body did not contain JSON
invalid_character Request body contains malformed JSON
too_large Request was too lare
internal_server_error Internal GoCardless server exception
api_key_not_active  API key is no longer active. You must re-issue a new one.
missing_authorization_header  Authorization header missing from request.
invalid_authorization_header  Authorization header has an invalid format. Should follow HTTP basic format: Authorization api_key_id:api_key_secret
api_key_not_found The API key id was not found.
missing_organisation  Organisation id was missing in resource links.
insufficient_permissions

### HTTP Status Code Summary
- 200 OK - Everything worked as expected.
- 400 Bad Request - Eg. invalid JSON.
- 401 Unauthorized - No valid API key provided.
- 402 Request Failed - Parameters were valid but request failed.
- 422 Validation Failed - Parameters were invalid.
- 404 Not Found - The requested item doesn't exist.
- 500, 502, 503, 504 Server errors - something went wrong on GoCardless end.

## What changes are considered “backwards-compatible”?
- Adding new API resources.
- Adding new optional request parameters to existing API methods.
- Adding new properties to existing API responses.
- Changing the order of properties in existing API responses.
- Changing the length or format of object IDs or other opaque strings.
- This includes adding or removing fixed prefixes (such as ch_ on charge - IDs).
- You can safely assume object IDs we generate will never exceed 128 characters, but you should be able to handle IDs of up to that length.
- If for example you’re using MySQL, you should store IDs in a VARCHAR(128) COLLATE utf8_bin column (the COLLATE configuration ensures case-sensitivity in lookups).
- Adding new event types. Your webhook listener should gracefully handle unfamiliar events types.

## Versioning changes
The versioning scheme is designed to promote incremental improvement to the API and discourage rewrites.

### Format:
Versions should be dated as ISO8601 (YYYY-MM-DD)
- Good: 2014-05-04
- Bad: v-1.1, v1.2, 1.3, v1, v2

### Version maintenance:
Maintain old API versions for at least 6 months.

### Implementation guidelines
When possible API versions should be tied to a set of API keys or the account in use.
API keys should maintain a last used date and the API version. Automated reminders should be set up to remind users to update their API version.

Optional header Format to specify version:
```
Accept: application/json; version=YYYY-MM-DD
Vary: Accept
```

Make sure `Vary` has the `Accept` header, or caching won't be invalidated by new versions.

See Caching.

## Pagination
All list/index endpoints must be paginated by default.
Pagination must be reverse chronological. To enable cursor based pagination, ids should be increasing.

Defaults:
`limit=50`
`after=NEWEST_RESOURCE`
`before=null`

Limits:
`limit=500`

Parameters:

## TABLE
Name  Type  Description
after string  id to start after
before  string  id to start before
limit numeric maximum number of records

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
    ...
  }]
}
```

## Schema
All API access is over HTTPS, and accessed from the api.gocardless.com domain. All data is sent and received as JSON.

## Validation
Only implement validation if it is strictly necessary. For example to support a multi-stage form where each step must be validated before being created.

To implement validation, resources create endpoint should accept a `dry_run` parameter. A successful validation should return a `200 OK`.

## Mock Responses
It is suggested that each resource accept a `mock` parameter on the testing server. Passing this parameter should return a mock data response (bypassing the backend).

Implementing this feature early in development ensures that the API will exhibit consistent behavior, supporting a test driven development methodology.
Note: If the mock parameter is included in a request to the production environment, an error should be raised.

Only support `mock` parameter in test environments.

## Mock Errors
To aid development it is recommended that each resource accepts a `raise_error_type` parameter on the testing server.

Passing this parameter would result in the error supplied being returned in the response.

Example:
```
GET /payments/pay_123?raise_error_type=invalid_request
{
  "message" :"Body should be a JSON Hash",
  "type": "invalid_request",
  "documentation_url": "https://developer.gocardless.com/v2/errors"
}
```

## Enveloping
Where you have limited access to http headers you can envelope the request.
A JSONP request is automatically enveloped.

To envelope a request set the param `?envelope_response=true`

Example:
```
{
  "status_code": "200",
  "headers": {
    "X-RateLimit-Limit": "5000",
    "X-RateLimit-Remaining": "4966",
    "X-RateLimit-Reset": "Thu, 01 Dec 1994 16:00:00 GMT"
  },
  "body": {
    // the data
  }
}
```

## JSONP
All endpoints should support JSONP and response enveloping.

You can send a ?callback parameter to any GET call to have the results wrapped in a JSON function. The response includes the same data output as the regular API, plus the relevant HTTP Header information.

To create a JSONP response set the param `?callback=callbackFunction`

Example:
```
// curl https://api.gocardless.com?callback=foo
/**/foo({
  "status_code": "200",
  "headers": {
    "X-RateLimit-Limit": "5000",
    "X-RateLimit-Remaining": "4966",
    "X-RateLimit-Reset": "Thu, 01 Dec 1994 16:00:00 GMT"
  },
  "body": {
    // the data
  }
})
```

## Handling PUT/PATCH/DELETE
If your http client or firewall doesn't support `PUT`, `PATCH` or `DELETE` you  can make a `POST` request with the header `X-HTTP-Method-Override: PATCH` or `X-HTTP-Method-Override: PUT`. The request must be a `POST`.

## Updates
Full/partial updates using PUT
`PUT` should replace any parameters passed and ignore fields not submitted.

Example:

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

Partial Updates using JSON PATCH
A `PATCH` should follow the JSON Patch spec. This enables patching nested documents.
Specification: http://tools.ietf.org/html/draft-ietf-appsawg-json-patch-10

```
PATCH /my/data HTTP/1.1
Host: example.org
Content-Length: 326
Content-Type: application/json-patch
If-Match: "abc123"

[
  { "op": "test", "path": "/a/b/c", "value": "foo" },
  { "op": "remove", "path": "/a/b/c" },
  { "op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ] },
  { "op": "replace", "path": "/a/b/c", "value": 42 },
  { "op": "move", "from": "/a/b/c", "path": "/a/b/d" },
  { "op": "copy", "from": "/a/b/d", "path": "/a/b/e" }
]
```

`path`  is a JSON Pointer: http://tools.ietf.org/html/draft-ietf-appsawg-json-pointer-07

## JSON encode POST, PUT & PATCH bodies
POST, PUT & PATCH expect JSON bodies in request. `Content-Type` header is required to be set to `application/json`.
For unsupported media types a 415 Unsupported Media Type response code is returned.

## Caching
Most responses return an ETag header. Many responses also return a Last-Modified header. You can use the values of these headers to make subsequent requests to those resources using the If-None-Match and If-Modified-Since headers, respectively. If the resource has not changed, the server will return a 304 Not Modified. Also note: making a conditional request and receiving a 304 response does not count against your rate limit, so we encourage you to use it whenever possible.

`Cache-Control: private, max-age=60`
`ETag: hash of contents`
`Last-Modified: updated_at`

Vary header
The following headers must be declared in Vary:
`Vary: Accept, Authorization, Cookie`

Any one of these headers can change the representation of the data and should invalidate a cached version. Users might be using different accounts to do admin, all with different privileges and resource visibility.
Accept can alter the returned representation.

Sources:
https://www.mnot.net/cache_docs/

## Compression
All responses should support gzip.

## Result filtering, sorting & searching
See JSON-API: http://jsonapi.org/format/#fetching-filtering

## Pretty printed responses
JSON responses should be pretty printed.

## Time zone / dates
Explicitly provide an ISO8601 timestamp with timezone information (Date time in UTC).
For API calls that allow for a timestamp to be specified, we use that exact timestamp.
These timestamps look something like 2014-02-27T15:05:06+01:00.  ISO 8601 UTC format: YYYY-MM-DDTHH:MM:SSZ.

## HTTP Rate Limiting
All endpoints must be rate limited.

You can check the returned HTTP headers of any API request to see your current rate limit status:

```
X-Rate-Limit-Limit: 5000
X-Rate-Limit-Remaining: 4994
X-Rate-Limit-Reset: Thu, 01 Dec 1994 16:00:00 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Retry-After: Thu, 01 May 2014 16:00:00 GMT

X-RateLimit-Reset uses the HTTP header date format: RFC 1123 (Thu, 01 Dec 1994 16:00:00 GMT)
```

Exceeding rate limit:
```
// 429 Too Many Requests
{
    "message": "API rate limit exceeded.",
    "type": "rate_limit_exceeded",
    "documentation_url": "http://developer.gocardless.com/#rate_limit_exceeded"
}
```

## CORS
Support Cross Origin Resource Sharing (CORS) for AJAX requests. You can read the CORS W3C working draft, or this intro from the HTML 5 Security Guide.

Any domain that is registered against the requesting account  is accepted.

```
$ curl -i https://api.gocardless.com -H "Origin: http://dvla.com"
HTTP/1.1 302 Found
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: ETag, Link, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes
Access-Control-Allow-Credentials: false

// CORS Preflight request
// OPTIONS 200
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-Modified-Since, If-None-Match, If-Unmodified-Since, X-Requested-With
Access-Control-Allow-Methods: GET, POST, PATCH, PUT, DELETE
Access-Control-Expose-Headers: ETag, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: false
```

## API SSL
All API request must be made over SSL. Any non-secure requests will return `api_error`. No redirects are made.

```
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
Set a request id header to aid debugging across services: `X-Request-Id` header.

## Client generated IDs
All create endpoints should accept a `client_id` field to be set.

This serves three purposes:
- Prevents replay attacks
- Enables idempotent resource creation, so client-side retries can occur
- Enables compound document creation
