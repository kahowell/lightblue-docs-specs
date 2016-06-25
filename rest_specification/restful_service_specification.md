# RESTful Service Specifications

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Goals

 - Uniformity between data-centric APIs and remote procedure APIs
 - Use of pre-existing standards/conventions when possible
 - Minimization of bike-shedding

Standards/Conventions used as inspiration:
 - [Whitehouse API Standards](https://github.com/WhiteHouse/api-standards)
  - RESTful URLs
 - [Spring Data REST](http://docs.spring.io/spring-data/rest/docs/current/reference/html/)
  - paging conventions
  - uses [JSON HAL](https://tools.ietf.org/html/draft-kelly-json-hal-08)
 - [JSend](http://labs.omniti.com/labs/jsend)
  - API response conventions

## Authentication

Authentication shall be done via TLS client certificates. The principal shall
be extracted from the client certificate of each request.

### Delegation

Implementations may support delegation as follows.

Principals with a special role (TBD), may send requests on behalf of any other
principal. The delegated principal shall be identified in the `Authorization`
HTTP header, using the `Bearer` auth-scheme (as defined in the
[OAuth 2.0 bearer token spec](http://tools.ietf.org/html/rfc6750#section-2.1)).

The contents of the `Bearer token` are up to implementation, but the `Bearer
token` must identify the delegated principal. That is to say, the
implementation may an OAuth 2.0 implementation, or the implemenation may be a
simpler one, for example, the implementation might just have the token be
the principal itself.

Example:

    Authorization: Bearer jdoe

## Authorization

Authorization shall be implemented on a per-endpoint basis.

If delegation is supported per above, then effective authorization shall be
determined by a logical AND; i.e. both the primary principal and the delegated
principal must have authorization in order for the request to be authorized.

## URIs

The APIs shall be versioned per entity.
The APIs shall be versioned with a major version only.
Multiple versions of an API may be supported.

The base URI for an entity must include a version.

The base URI for a given resource type is based on the *singular* name of the
resource and the API version. Example:

    https://example.com/rest/user/v1

## Operations Against a Resource Type

The following operations may be supported:

| HTTP Operation | Description            | Query Parameters                | Request Body | Response Body  |
|----------------|------------------------|---------------------------------|--------------|----------------|
| `POST`         | Create new resource(s) | -                               | JSON array   | -              |
| `PUT`          | Replace resource(s)    | -                               | JSON array   | -              |
| `PATCH`        | Update Resource(s)     | -                               | JSON array   | -              |
| `GET`          | List resource(s)       | `fields`,`page`,`size`,`sort[]` | -            | paginated list |
| `DELETE`       | Delete ALL resources   | -                               | -            | -              |

Endpoints may not support all methods (i.e. a service may provide read-only
access to a resource, or a service may not support certain operations for
performance or other considerations; for example, `GET` may not be supported
for `user` if listing all users is disallowed for performance reasons.

## Operations Against Specific Resources
Specific resources will be addressed through the base URI plus the identifier
for that resource by one or more field names, as many as required to identify
a single entity.

    /user/v1?id={id}
    /product/v1?sku=FOO&opUnit=100

If a resource is identified by a single field, then a single path parameter
must be allowed for brevity and familiarity.

    /user/v1/{id}

Failure to include enough fields to identify a single resource is a client 
error. The error response must include what fields are missing (TODO: 
document errors in non-function responses). To query for many results, see 
Additional Functions.

The following operations may be supported:

| HTTP Operation | Description                   | Query Parameters | Request Body | Response Body |
|----------------|-------------------------------|------------------|--------------|---------------|
| `GET`          | Retrieve the resource         | `fields`         | -            | JSON object   |
| `PUT`          | Replace the existing resource | -                | JSON object  | -             |
| `PATCH`        | Update an existing resource   | -                | JSON object  | -             |
| `DELETE`       | Delete the resource           | -                | -            | -             |

Clients should use the `fields` parameter for better performance.

Implementors should consider breaking out field represented by a JSON array
into sub-entities if there are performance concerns around its size. This
consideration should happen regardless of how the data is stored.

## Pagination

Paging shall be implemented via `page` and `size` query parameters. If none are
provided, the default page size shall be used.

The default page-size is implementation-specific.

| Parameter | Description               |
|-----------|---------------------------|
| `page`    | page index, starting at 0 |
| `size`    | page size                 |

## Sorting

Sorting shall be provided via the `sort[]` query parameter. `sort[]` may be
specified more than once in order to sort by more than one field. `sort[]` values
shall be of the form `sort[]={field}(,{direction})` where `direction` is `asc` or
`desc`. If `direction` is not specified, it will default to `asc`.

### Responses

The response shall be a JSON object formatted according to Spring Data REST
conventions.Specifically, the list is returned as a pluralized field name
under `_embedded`, and a `page` object supplies paging info. Example:

```javascript
GET /user

{
  "_embedded": {
    "users": [
      {
        // ...
      },
      // ...
    ]
  },
  "page": {
    "size": 20,
    "totalElements": 200,
    "totalPages": 10,
    "number": 0
  }
}
```

## Additional Functions

Additional APIs may expose other functionality related to a resource type.
Examples include:
 - query for a specific subset of resources
 - perform an action against a set of resources

Additional functions shall be `POST` only, and parameters will be provided in
the request body. The request body SHOULD be a JSON object with fields
corresponding to parameter names.
Examples:

```json
POST /user/v1/findByLogin

{
  "login": "bob"
}
```

Each function that can be performed against a specific resource must exist in
two forms:
 - non-batch
 - batch

The non-batch form shall take any relevant parameters in a JSON object. Example:

```json
POST /user/v1/deactivate

{
  "id": 123,
  "reason": "Banned"
}
```

The batch form shall take a JSON array of objects with the same schema. 
Example:

```json
POST /user/v1/deactivate

[
  {
    "id": 123,
    "reason": "Banned"
  },
  {
    "id": 456,
    "reason": "Banned"
  }
]
```

Responses for functions shall follow [JSend](http://labs.omniti.com/labs/jsend)
conventions:
 - Responses must be JSON objects
 - `status` field shall tell whether the function call was successful
  - possible values: `success`, `fail`, `error`
 - `data` shall have any feedback from the function call (function-specific)
 - `message` shall contain a descriptive error message in the case of an error.

Success example:
```json
{
  "status" : "success",
  "data" : null
}
```

Related resources may be exposed under additional endpoints off a 
resource-specific endpoint. Example:

    /user/v1/{id}/posts{&fields,page,size,sort[]}

## Future/Optional Features

In general, clients must ignore any fields that they don't expect. This allows
backward-compatibility by additional fields.

### Discoverability (HATEOAS)

Per HAL/Spring Data REST conventions, an implementation may enable usage by
having relationships between entities and APIs exposed via `_links` fields.

Example:
```javascript
{
  "_links": {
    "self": {
      "href": "https://example.com/rest/v1/user{&sort[],page,size}",
      "templated": true
    },
    "next": {
      "href": "https://example.com/rest/v1/user?page=1&size=20{&sort[]}",
      "templated": true
    }
  },
  "_embedded": {
    "users": [
      {
        // ...
      },
      // ...
    ]
  },
  "page": {
    "size": 20,
    "totalElements": 200,
    "totalPages": 10,
    "number": 0
  }
}
```
Given examples, links which might be added include:

| Name        | Description               |
|-------------|---------------------------|
| self        | the URI for this resource |
| prev        | previous paginated page   |
| next        | next paginated page       |
| posts       | link to posts for a user  |
| findByLogin | find user(s) by login     |

As illustrated, links may be for related entities, pagination, relevant
functions, etc.
