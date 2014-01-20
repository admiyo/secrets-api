OpenStack Secrets API v3
=========================

The Identity API primarily fulfills Secret storage, retrieval, and verification
needs within OpenStack, and is intended to provide a programmatic facade in
front of existing PKI and Symmetric Key management front ends.



What's New in Version 1.0
-------------------------

These features are considered stable as of April 1, 2014.


Document Overview
-----------------

This document is always evolving as new features are added, use cases are
clarified, etc. API features added since the original publication of this
document are summarized above, grouped by the API version in which they were
first introduced. This document is treated as the single source of truth for
the entire 1.x series of the API.

A particular implementation of the API is never referenced by name. This is
documentation for the HTTP API itself. Details of the code-base servicing the
API, such as architecture, configuration, and deployment, are not relevant
here.

The "API Conventions" section defines architectural patterns applied to the
entire API until a portion of the API documents an exception to the overall
conventions. Details of the conventions are not repeated throughout the
document, except in examples, so the reader is expected to have understood the
conventions before reading any further. The goal is to reduce the cost of
documentation maintenance (DRY) and improve self-consistency across the API,
which makes the API more intuitive to readers and fosters simpler
implementations.

A high level overview of the resources presented by the API are documented in
the "API Resources" section, including required and optional attributes, use
cases and expected behaviors. Specific API calls are not enumerated, although
the feature set of the related calls should be described if it deviates from
the conventions used by the rest of the API (for example, a resource could be
constrained as "a read-only collection").

Finally, the specific calls supported by the API are enumerated with examples
at the end of the document. The examples are intended to be "realistic"
representations of actual requests and responses you could expect from an
implementation of the API. Specifically, the JSON should be syntactically valid
and use data that is self-consistent with related calls.

The features described by this document are intended to be applicable to all
implementations of the API. If a particular implementation or deployment should
not be expected to have a use case for a particular feature, that feature
should be documented as an extension to this API. Extensions may suffix
existing resources with their own namespace in order to add new resources, or
prefix new attributes on existing resource representations. To clearly
distinguish extensions from the core API (which is described by this document)
and avoid namespace collisions between extensions, suffixes and prefixes are
composed of an uppercased abbreviation of the organization supporting the
extension (such as "OS" for OpenStack), followed by a hyphen ("-"), followed by
an uppercased abbreviation of the extension name (such as "OAUTH1" for OAuth
1.0). Therefore, an extension could be identified as "OS-OAUTH1".

API Conventions
---------------

This section describes architectural patterns applied throughout the Identity
API, unless an exception to these conventions is specifically documented. In
general, the Identity API provides an HTTP interface using JSON as the primary
transport format.

Each resource contains a canonically unique identifier (ID) defined by the
Identity service implementation and is provided as the `id` attribute; Resource
ID's are strings of non-zero length.

The resource paths of all collections are plural and are represented at the
root of the API (e.g. `/v1/policies`).

TCP port 35357 is designated by the Internet Assigned Numbers Authority
("IANA") for use by OpenStack Identity services. Example API requests &
responses in this document therefore assume that the Identity service
implementation is deployed at the root of `http://identity:35357/`.

### Headers

- `X-Auth-Token`

  This header is used to convey the API user's authentication token when
  accessing Identity APIs.

- `X-Subject-Token`

  This header is used to convey the subject of the request for token-related
  operations.

### Required Attributes

For collections:

- `links` (object)

  Specifies a list of relational links to the collection.

  - `self` (url)

    A self-relational link provided as an absolute URL. This attribute is
    provided by the identity service implementation.

  - `previous` (url)

    A relational link to the previous page of the list, provided as an absolute
    URL. This attribute is provided by the identity service implementation. May
    be null.

  - `next` (url)

    A relational to the next page of the list, provided as an absolute URL.
    This attribute is provided by the identity service implementation. May be
    null.

For members:

- `id` (string)

  Globally unique resource identifier. This attribute is provided by the
  identity service implementation.

- `links` (object)

  Specifies a set of relational links relative to the collection member.

  - `self` (url)

    A self-relational link provided as an absolute URL. This attribute is
    provided by the identity service implementation.

### CRUD Operations

Unless otherwise documented (tokens being the notable exception), all resources
provided by the Identity API support basic CRUD operations (create, read,
update, delete).

The examples in this section utilize a resource collection of Entities on
`/v1/entities` which is not actually a part of the Identity API, and is used
for illustrative purposes only.

#### Create an Entity

When creating an entity, you must provide all required attributes (except those
provided by the Identity service implementation, such as the resource ID):

Request:

    POST /entities

    {
        "entity": {
            "name": string,
            "description": string,
            "enabled": boolean
        }
    }

The full entity is returned in a successful response (including the new
resource's ID and a self-relational link), keyed by the singular form of the
resource name:

    201 Created

    {
        "entity": {
            "id": string,
            "name": string,
            "description": string,
            "enabled": boolean,
            "links": {
                "self": url
            }
        }
    }

#### List Entities

Request the entire collection of entities:

    GET /entities

A successful response includes a list of anonymous dictionaries, keyed by the
plural form of the resource name (identical to that found in the resource URL):

    200 OK

    {
        "entities": [
            {
                "id": string,
                "name": string,
                "description": string,
                "enabled": boolean,
                "links": {
                    "self": url
                }
            },
            {
                "id": string,
                "name": string,
                "description": string,
                "enabled": boolean,
                "links": {
                    "self": url
                }
            }
        ],
        "links": {
            "self": url,
            "next": url,
            "previous": url
        }
    }

##### List Entities filtered by attribute

Beyond each resource's canonically unique identifier (the `id` attribute), not
all attributes are guaranteed unique on their own. To list resources which match
a specified attribute value, we can perform a filter query using a query string
with one or more attribute/value pairs:

    GET /entities?name={entity_name}&enabled={entity_enabled}

The response is a subset of the full collection:

    200 OK

    {
        "entities": [
            {
                "id": string,
                "name": string,
                "description": string,
                "enabled": boolean,
                "links": {
                    "self": url
                }
            }
        ],
        "links": {
            "self": url,
            "next": url,
            "previous": url
        }
    }

#### Get an Entity

Request a specific entity by ID:

    GET /entities/{entity_id}

The full resource is returned in response:

    200 OK

    {
        "entity": {
            "id": string,
            "name": string,
            "description": string,
            "enabled": boolean,
            "links": {
                "self": url
            }
        }
    }

##### Nested collections

An entity may contain nested collections, in which case the required attributes
for collections still apply; however, to avoid conflicts with other required
attributes, the required attributes of the collection are prefixed with the
name of the collection. For example, if an `entity` contains a nested
collection of `objects`, the `links` for the collection of `objects` is called
`objects_links`:

    {
        "entity": {
            "id": string,
            "name": string,
            "description": string,
            "enabled": boolean,
            "links": {
                "self": url
            },
            "objects": [
                {
                    "id": string,
                    "name": string,
                    "description": string,
                    "enabled": boolean,
                    "links": {
                        "self": url
                    }
                }
            ],
            "objects_links": {
                "self": url,
                "next": url,
                "previous": url
            }
        }
    }

#### Update an Entity

Partially update an entity (unlike a standard `PUT` operation, only the
specified attributes are replaced):

    PATCH /entities/{entity_id}

    {
        "entity": {
            "description": string
        }
    }

The full entity is returned in response:

    200 OK

    {
        "entity": {
            "id": string,
            "name": string,
            "description": string,
            "enabled": boolean,
            "links": {
                "self": url
            }
        }
    }

#### Delete an Entity

Delete a specific entity by ID:

    DELETE /entities/{entity_id}

A successful response does not include a body:

    204 No Content

### HTTP Status Codes

The Identity API uses a subset of the available HTTP status codes to
communicate specific success and failure conditions to the client.

#### 200 OK

This status code is returned in response to successful `GET` and `PATCH`
operations.

#### 201 Created

This status code is returned in response to successful `POST` operations.

#### 204 No Content

This status code is returned in response to successful `HEAD`, `PUT` and
`DELETE` operations.

#### 300 Multiple Choices

This status code is returned by the root identity endpoint, with references to
one or more Identity API versions (such as ``/v1/``).

#### 400 Bad Request

This status code is returned when the Identity service fails to parse the
request as expected. This is most frequently returned when a required attribute
is missing, a disallowed attribute is specified (such as an `id` on `POST` in a
basic CRUD operation), or an attribute is provided of an unexpected data type.

The client is assumed to be in error.

#### 401 Unauthorized

This status code is returned when either authentication has not been performed,
the provided X-Auth-Token is invalid or authentication credentials are invalid
(including the user, project or domain having been disabled).

#### 403 Forbidden

This status code is returned when the request is successfully authenticated but
not authorized to perform the requested action.

#### 404 Not Found

This status code is returned in response to failed `GET`, `HEAD`, `POST`,
`PUT`, `PATCH` and `DELETE` operations when a referenced entity cannot be found
by ID. In the case of a `POST` request, the referenced entity may be in the
request body as opposed to the resource path.

#### 409 Conflict

This status code is returned in response to failed `POST` and `PATCH`
operations. For example, when a client attempts to update an entity's unique
attribute which conflicts with that of another entity in the same collection.

Alternatively, a client should expect this status code when attempting to
perform the same create operation twice in a row on a collection with a
user-defined and unique attribute. For example, a User's `name` attribute is
defined to be unique and user-defined, so making the same ``POST /users``
request twice in a row will result in this status code.

The client is assumed to be in error.

#### 500 Internal Server Error

This status code is returned when an unexpected error has occurred in the
Identity service implementation.

#### 501 Not Implemented

This status code is returned when the Identity service implementation is unable
to fulfill the request because it is incapable of implementing the entire API
as specified.

For example, an Identity service may be incapable of returning an exhaustive
collection of Projects with any reasonable expectation of performance, or lack
the necessary permission to create or modify the collection of users (which may
be managed by a remote system); the implementation may therefore choose to
return this status code to communicate this condition to the client.

#### 503 Service Unavailable

This status code is returned when the Identity service is unable to communicate
with a backend service, or by a proxy in front of the Identity service unable
to communicate with the Identity service itself.

API Resources
-------------

### Secrets: `/v1/secrets`

### Orders: `/v1/secrets`

### Verifications: `/v1/secrets`





Core API
--------


### Secrets

#### Create secret: `POST /secrets`

Request:

    {
    }

Response:

    Status: 201 Created

    {
    }

#### List secrets: `GET /secrets`

query filter for "domain_id", "email", "enabled", "name" (optional)

Response:

    Status: 200 OK

    {
    }

#### Get secret: `GET /secrets/{secret_id}`

Response:

    Status: 200 OK

    {
    }



#### Update secret: `PATCH /secrets/{secret_id}`


The request block ....

Response:

    Status: 200 OK

    {
    }

#### Delete secret: `DELETE /secrets/{secret_id}`

Response:

    Status: 204 No Content

### Orders

#### Create order: `POST /orders`

Request:

    {
    }

Response:

    Status: 201 Created

    {
    }

#### List orders: `GET /orders`

query filter for "domain_id", "email", "enabled", "name" (optional)

Response:

    Status: 200 OK

    {
    }

#### Get order: `GET /orders/{order_id}`

Response:

    Status: 200 OK

    {
    }



#### Update order: `PATCH /orders/{order_id}`


The request block ....

Response:

    Status: 200 OK

    {
    }

#### Delete order: `DELETE /orders/{order_id}`

Response:

    Status: 204 No Content


### Verifications

#### Create verification: `POST /verifications`

Request:

    {
    }

Response:

    Status: 201 Created

    {
    }

#### List verifications: `GET /verifications`

query filter for "domain_id", "email", "enabled", "name" (optional)

Response:

    Status: 200 OK

    {
    }

#### Get verification: `GET /verifications/{verification_id}`

Response:

    Status: 200 OK

    {
    }



#### Update verification: `PATCH /verifications/{verification_id}`


The request block ....

Response:

    Status: 200 OK

    {
    }

#### Delete verification: `DELETE /verifications/{verification_id}`

Response:

    Status: 204 No Content



