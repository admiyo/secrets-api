This wiki page details the API for the latest Barbican release. In particular:
* [Endpoints & Versioning][end] - Brief discussion about Barbican's approach to service URIs.
* [Secrets Resource][sec] - Details storing, retrieving and deleting secrets.
* [Orders Resource][ord] - Details the ordering facilities for Barbican, used to generate secrets asynchronously.
* [Verifications Resource][ver] - Details the verification request processing facilities for Barbican, used to perform verification analysis on specific cloud resources.
* [Examples][exmp] - Provides specific examples utilizing the secrets and orders API.

[end]: https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#endpoints--versioning
[sec]: https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#secrets-resource
[ord]: https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#orders-resource
[ver]: https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#verifications-resource
[exmp]: https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#examples

# Endpoints & Versioning

The barbican service is assumed to be hosted on an SSL enabled, geographically labelled endpoint. An example for a valid URI might look like the following:

    https://dfw.secrets.api.rackspacecloud.com

Versioning will be achieved through a URI constant element, as shown in the example below:

    https://dfw.secrets.api.rackspacecloud.com/v1/

# Secrets Resource
The secrets resource is the heart of the Barbican service. It provides access to the secret / keying material stored in the system. 

The secret scheme represents the actual secret or key that will be presented to the application. Secrets can be of any format, but additional functionality may be available for known types of symmetric or asymmetric keys. The schema for creating a secret differs from the retrieval schema is shown next. 

## Creating Secrets
Secrets can be created in two ways: Via POST including a payload, or with a POST without a payload followed by a PUT. 

Note that the POST calls create secret _metadata_. If the payload is provided with the POST call, then it is encrypted and stored, and then linked with this metadata. Otherwise, the PUT call provides this payload. Hence clients must provide the secret information to store via these 'create' operations. This should not be confused with the secret 'generation' process via the `orders` resource below, whereby Barbican generates the secret information.

### POST
Below is an example of a `secret` POST request that includes a payload.

```javascript
POST v1/{tenant_id}/secrets

{
  "name": "AES key",
  "expiration": "2014-02-28T19:14:44.180394",
  "algorithm": "aes",
  "bit_length": 256,
  "mode": "cbc",
  "payload": "gF6+lLoF3ohA9aPRpt+6bQ==",
  "payload_content_type": "application/octet-stream",
  "payload_content_encoding": "base64"
}

```

Where:
* **name** - (optional) Human readable name for the secret. If a name is not supplied, the UUID will be displayed for this field on subsequent GET calls (see below).
* **expiration** - (optional) The expiration date for the secret in ISO-8601 format. Once the secret has expired, it will no longer be returned by the API or agent. If this field is not supplied, then the secret has no expiration date.
* **algorithm** - (optional) The algorithm type used to generate the secret.
* **bit_length** - (optional) The bit-length of the secret. Must be a positive integer.
* **mode** - (optional) The type/mode of the algorithm associated with the secret information.
* **payload** - (optional) The secret's unencrypted plain text. If provided, this field's value must be non-empty, and you must also provide the `payload_content_type`. This field can be omitted allowing for the secret information to be provided via a subsequent PUT call (see below)._
* **payload_content_type** - (optional) The type/format the secret data is provided in. Required if `payload` is specified._  Supported values are:
    * _"text/plain"_ - Used to store plain text secrets. 
        * Other options are _"text/plain; charset=utf-8"_. If charset is omitted, utf-8 will be assumed. 
        * Note that some types are normalized before being stored as secret meta-data, such as converting "text/plain; charset=utf-8" to "text/plain". _Hence retrieved meta-data may not exactly match what was specified in the POST or PUT calls_.
        * payload_content_encoding must **not** be specified when payload_content_type is _"text/plain"_
    * _"application/octet-stream"_ - Used to store binary secrets from a base64 encoded payload.  If this value is used, you must also include the content encoding.
* **payload_content_encoding** - (optional) The _encoding_ format used to provide the payload data. Barbican might translate and store the secret data into another format. _Required if `payload_content_type` is "application/octet-stream"._  Supported values are:
    * _"base64"_ - Used to specify base64 encoded payloads.

If the `payload` is not provided, only the secret metadata will be retrievable from Barbican and any attempt to retrieve decrypted data for that secret will fail. Deferring the secret information to a PUT request is useful for secrets that are in binary format and are not suitable for base64 encoding.

If the POST call succeeds, a URI to the new secret will be provided such as per the example below:

```javascript
{
    "secret_ref": "http://localhost:9311/v1/{tenant_id}/secrets/a8957047-16c6-4b05-ac57-8621edd0e9ee"
}
```

### PUT
To provide secret information after the secret is created, clients would send a PUT request to the secret URI. Note that **a PUT request can only be performed once after a POST call that does not include a payload**.  Also note that no other attributes of a secret can be modified via PUT after it is POST-ed.

The PUT request should include the appropriate Content-Type and Content-Encoding definitions. (see Examples below for more information.)

## Retrieving Secrets
Secrets are comprised of metadata about the secret (algorithm, bit-length, etc.) and encrypted data associated with the secret. Hence the API supports retrieving either a secret's metadata or its decrypted data.

### GET - Individual Secret - Metadata Only
GET requests for a `secret` with an `Accept` header set to `application/json` will return a response such as the one below. Only metadata about the secret is returned, rather than the decrypted secret information itself. This allows for a more rapid response for large secrets, or large lists of secrets, as well as accommodating multi-part secrets such as SSL certificates, which may have both a public and private key portions that could be individually retrieved.

An example GET call for an individual secret is below.

```javascript
GET v1/{tenant_id}/secrets/888b29a4-c7cf-49d0-bfdf-bd9e6f26d718

{
  "status": "ACTIVE",
  "updated": "2013-06-28T15:23:33.092660",
  "name": "AES key",
  "algorithm": "AES",
  "mode": "cbc",
  "bit_length": 256,
  "content_types": {
    "default": "application/octet-stream"
  },
  "expiration": "2013-05-08T16:21:38.134160",
  "secret_ref": "http://localhost:8080/v1/12345/secrets/888b29a4-c7cf-49d0-bfdf-bd9e6f26d718",
}
```

Where:
* **name** - Human readable name for the secret. If a name was not provided during the POST call above, then its UUID is returned.
* **algorithm** - See POST example above.
* **mode** - See POST example above.
* **bit_length** - See POST example above.
* **content_types** - Available content mime types for the format of the decrypted secret. For example, SSL certificates may have both 'public' (for public key) and 'private' (for private key) types available. 
    * _This value is only shown if a secret has encrypted data associated with it._
    * Note that some types are normalized, such as converting "text/plain; charset=utf-8" to "text/plain". _Hence retrieved meta-data may not exactly match what was specified in the POST or PUT calls_.
* **expiration** - UTC time when the secret will expire.  Attempting to retrieve a secret after the expiration date will result in an error.
* **secret_ref** - Self URI to the secret.

### GET - List of Secrets Per Tenant
Performing a GET on the secrets resource with no UUID retrieves a batch of the N most recent secrets (metadata only) per the requesting tenant, as per the example response below. The `limit` and `offset` parameters are used to control pagination of N secrets, as described after this example.

```javascript
http://localhost:9311/v1/{tenant_id}/secrets?limit=3&offset=2

{
  "secrets": [
    {
      "status": "ACTIVE",
      "updated": "2013-06-28T15:23:30.668641",
      "mode": "cbc",
      "name": "Main Encryption Key",
      "algorithm": "AES",
      "created": "2013-06-28T15:23:30.668619",
      "secret_ref": "http://localhost:9311/v1/12345/secrets/e171bb2d-f14f-433e-84f0-3dfcac7a7311",
      "expiration": "2014-06-28T15:23:30.668619",
      "bit_length": 256,
      "content_types": {
        "default": "application/octet-stream"
      }
    },
    {
      "status": "ACTIVE",
      "updated": "2013-06-28T15:23:32.210474",
      "mode": "cbc",
      "name": "Backup Key",
      "algorithm": "AES",
      "created": "2013-06-28T15:23:32.210467",
      "secret_ref": "http://localhost:9311/v1/12345/secrets/6dba7827-c232-4a2b-8f3d-f523ca3a3f99",
      "expiration": null,
      "bit_length": 256,
      "content_types": {
        "default": "application/octet-stream"
      }
    },
    {
      "status": "ACTIVE",
      "updated": "2013-06-28T15:23:33.092660",
      "mode": null,
      "name": "PostgreSQL admin password",
      "algorithm": null,
      "created": "2013-06-28T15:23:33.092635",
      "secret_ref": "http://localhost:9311/v1/12345/secrets/6dfa448d-c35a-4158-abaf-e4c249efb580",
      "expiration": null,
      "bit_length": null,
      "content_types": {
        "default": "text/plain"
      }
    }
  ],
  "next": "http://localhost:9311/v1/12345/secrets?limit=3&offset=5",
  "previous": "http://localhost:9311/v1/12345/secrets?limit=3&offset=0"
}
```

The retrieved list of secrets is ordered by oldest to newest `created` date. The URL parameters (`?limit=3&offset=2` in this example) provide a way to window or page the retrieved list, with the `offset` representing how many records to skip before retrieving the list, and the `limit` representing the maximum number of records retrieved (up to `100`). If the parameters are not provided, then up to `10` records are retrieved. 

To access any records before the retrieved list, the 'previous' link is provided. The 'next' link can be used to retrieve records after the current list.

### GET - Decrypted Secret Data

To retrieve the decrypted secret information, perform a GET with the `Accept` header set to one of the `content_types` specified in the GET metadata call. Note that even if a binary secret is provided in the `base64` format, it is converted to binary by Barbican prior to encryption and storage. _Thereafter the secret will only be decrypted and returned as raw binary._ See examples below for more info.

## Secrets Summary
> https://.../v1/{tenant_id}/secrets

| Method | Description |
|--------|-------------|
| GET    | Allows a user to list all secrets in a tenant. _Note: the actual secret data will not be listed here. Clients must instead make a separate call to get the secret details to view the secret._ |
| POST   | Allows a user to create a new secret. This call expects the user to provide a secret. To have the API generate a secret, see the `orders` API below. Returns 201 if the secret has been created. |

> https://.../v1/{tenant_id}/secrets/{secret_uuid}/

| Method | Description |
|--------|-------------|
| GET    | Gets the information for the specified secret. For the `application/json` accept type, only metadata about the secret is returned. If one of the 'content_types' accept types is specified instead, that portion of the secret will be decrypted and returned. |
| PUT    | Allows the user to upload secret data for a specified secret _(if the secret does not already have data associated with it)_. Returns 200 on a successful request. |
| DELETE | Deletes the secret. |

### Error Responses

| Action | Error Code | Notes |
|--------|------------|-------|
| POST secret with invalid data | 400 | Can include schema violations such as mime-type not specified |
| POST secret with 'payload' empty | 400 | The 'payload' JSON attribute was provided, but no value was assigned to it. |
| POST secret with 'payload' too large | 413 | Current size limit is 10,000 bytes |
| POST secret with 'payload_content_type' not supported | 400 | Caused when no crypto plugin supports the payload_content_type requested |
| GET secret that doesn't exist | 404 | The supplied UUID doesn't match a secret in the data store |
| GET secret with unsupported Accept | 406 | The secret data cannot be retrieved in the requested Accept header mime-type |
| GET secret (non-JSON) with no associated encrypted data | 404 | The secret metadata has been created, but the encrypted data for it has not yet been supplied, hence cannot be retrieved via a non 'application/json' mime type |
| PUT secret that doesn't exist | 404 | The supplied UUID doesn't match a secret in the data store for the given tenant |
| PUT secret with unsupported Content-Type | 400 | Caused when no crypto plugin supports the payload_content_type requested in the Content-Type |
| PUT secret that already has encrypted data | 409 | Secret already has encrypted data associated with it |
| PUT secret with empty 'payload' data | 400 | No value was provided in the payload |
| PUT secret with too large 'payload' data | 413 | Current size limit is 10,000 bytes for uploaded secret data |
| DELETE secret that doesn't exist | 404 | The supplied UUID doesn't match a secret in the data store |


# Orders Resource

The ordering resource allows for the generation of secret material by Barbican. The ordering object encapsulates the workflow and history for the creation of a secret. This interface is implemented as an asynchronous process since the time to generate a secret can vary depending on the type of secret. 

## POST
An example of an `orders` POST request is below.

```javascript
POST v1/{tenant_id}/orders

{
  "secret": {
    "name": "secretname",
    "algorithm": "AES",
    "bit_length": 256,
    "mode": "cbc",
    "payload_content_type": "application/octet-stream"
  }
}
```

Where the elements of the `secret` element match those of the `secret` POST request above, but without the `payload` attributes. 

## PUT

Currently nothing can be edited in an order.


## GET - Individual Order
GET requests for an `order` will return a response such as in the example below. 

```javascript
GET v1/{tenant_id}/orders/{order_id}

{
  "secret": {
    "name": "secretname",
    "algorithm": "aes",
    "bit_length": 256,
    "mode": "cbc",
    "payload_content_type": "application/octet-stream"
  },
  "order_ref": "http://localhost:8080/v1/12345/orders/f9b633d8-fda5-4be8-b42c-5b2c9280289e",
  "secret_ref": "http://localhost:8080/v1/12345/secrets/888b29a4-c7cf-49d0-bfdf-bd9e6f26d718",
  "status": "ERROR",
  "error_status_code": "400 Bad Request",
  "error_reason": "Secret creation issue seen - content-encoding of 'bogus' not supported."
}

```

Where:
* **secret** - Secret parameters provided in the original order request. _Note that this is not the same as retrieving a Secret resource per the `secrets` resource, so elements such as a secret's `content_types` will not be displayed. To see such details, perform a GET on the `secret_ref`._
* **order_ref** - URI to this order.
* **status** - Status of the order, one of PENDING, ACTIVE or ERROR. Clients should poll the order for a status change to ACTIVE (in which case `secret_ref` has the secret details) or ERROR (in which case `error_reason` has the error reason, and 'error_status_code' has an HTTP-style status code).
* **secret_ref** - URI to the secret *once it is generated*. This field is not available unless the status is ACTIVE.
* **error_status_code** - (optional) HTTP-style status code of the root cause error condition, only if status is ERROR.
* **error_reason** - (optional) Details of the root cause error condition, only if status is ERROR.

## GET - List of Orders Per Tenant
Performing a GET on the secrets resource with no UUID retrieves a batch of the N most recent orders per the requesting tenant, as per the example response below. The `limit` and `offset` parameters function similar to the GET secrets list detailed above. 

```javascript
http://localhost:9311/v1/12345/orders?limit=3&offset=2

{
  "orders": [
    {
      "status": "ACTIVE",
      "secret_ref": "http://localhost:9311/v1/12345/secrets/bf2b33d5-5347-4afb-9009-b4597f415b7f",
      "updated": "2013-06-28T18:29:37.058718",
      "created": "2013-06-28T18:29:36.001750",
      "secret": {
        "name": "secretname",
        "algorithm": "aes",
        "bit_length": 256,
        "mode": "cbc",
        "payload_content_type": "application/octet-stream"
      },
      "order_ref": "http://localhost:9311/v1/12345/orders/3100078a-6ab1-4c3f-ab9f-295938c91733"
    },
    {
      "status": "ACTIVE",
      "secret_ref": "http://localhost:9311/v1/12345/secrets/fa71b143-f10e-4f7a-aa82-cc292dc33eb5",
      "updated": "2013-06-28T18:29:37.058718",
      "created": "2013-06-28T18:29:36.001750",
      "secret": {
        "name": "secretname",
        "algorithm": "aes",
        "bit_length": 256,
        "mode": "cbc",
        "payload_content_type": "application/octet-stream"
      },
      "order_ref": "http://localhost:9311/v1/12345/orders/30b3758a-7b8e-4f2c-b9f0-f590c6f8cc6d"
    }
  ]
}
```

The retrieved list of orders is ordered by oldest to newest `created` date.

## Orders Summary
> https://.../v1/{tenant_id}/orders/

| Method | Description |
|--------|-------------|
| GET    | Returns a list of all orders for a customer. |
| POST   | Starts the process of creating a secret. This call will return immediately with a 202 OK and a link to the detail order object (see below). |

> https://.../v1/{tenant_id}/orders/{order_uuid}

| Method | Description |
|--------|-------------|
| GET    | Returns the detailed order data including a link to the secret generated as a result of the order (if available). |
| PUT    | **Not yet supported**. Allows the editing of an order where allowed. |
| DELETE | Cancels an order. |

## Error Responses

| Action | Error Code | Notes |
|--------|------------|-------|
| POST order with invalid data | 400 | Can include schema violations such as the secret's mime-type not specified |
| POST secret with 'payload_content_type' not supported | 400 | Caused when no crypto plugin supports the payload_content_type requested |
| GET order that doesn't exist | 404 | The supplied UUID doesn't match a order in the data store |
| DELETE order that doesn't exist | 404 | The supplied UUID doesn't match a order in the data store |


# Verifications Resource

**NOTE: The final request and response parameters for this call is a work in progress.**

The verifications resource allows for verifying a cloud resource (such as an image) via Barbican. The verification object encapsulates the workflow and history for the creation of a verification process. This interface is implemented as an asynchronous process since external services might need to be queried to complete the verification process.

## POST
An example of a `verifications` POST request is below.

```javascript
POST v1/{tenant_id}/verifications

{
  "resource_type": "image",
  "resource_ref": "(resource URI here)",
  "resource_action": "vm_attach",
  "impersonation_allowed": false
}
```
Where:
* **resource_type** - Type of resource to verify. Choices include: image.
* **resource_ref** - URI or ID to the resource to verify.
* **resource_action** - Proposed action for the resource.
* **impersonation_allowed** - Indicate if user impersonation of resource action is allowed.

The following is an example response if the POST call is successful:

```bash
{
    "verification_ref": "http://localhost:9311/v1/1234/verifications/233d7d8c-db8c-427b-867e-a3ddfc03a8ab"
}
```

## PUT

Currently nothing can be edited in a verification resource.


## GET - Individual Verification
GET requests for a `verification` will return a response such as in the example below. 

```javascript
GET v1/{tenant_id}/verifications/{verification_id}

{
    "status": "ACTIVE",
    "updated": "2013-11-27T17:45:15.132732",
    "created": "2013-11-27T17:45:15.084483",
    "resource_action": "vm_attach",
    "resource_ref": "http://yada.com",
    "impersonation_allowed": true,
    "verification_ref": "http://iad-int-api.cloudkeep.io:9311/v1/12345/verifications/b02f668e-8ccf-46a9-ba47-ec03b2233e4f",
    "is_verified": true,
    "resource_type": "image"
}
```

Where:
* **verification_ref** - URI to this verification request.
* **status** - Status of the verification, one of PENDING, ACTIVE or ERROR. Clients should poll the verification for a status change to ACTIVE (in which case `is_verified` has the final result) or ERROR (in which case `error_reason` has the error reason, and 'error_status_code' has an HTTP-style status code).
* **error_status_code** - (optional) HTTP-style status code of the root cause error condition, only if status is ERROR.
* **error_reason** - (optional) Details of the root cause error condition, only if status is ERROR.

## GET - List of Verifications Per Tenant
Performing a GET on the verifications resource with no UUID retrieves a batch of the N most recent verifications per the requesting tenant, as per the example response below. The `limit` and `offset` parameters function similar to the GET orders list detailed above. 

```javascript
http://localhost:9311/v1/12345/verifications?limit=3&offset=2

{
  "total": 2,
  "verifications": [
    {
        "status": "ACTIVE",
        "updated": "2013-11-27T17:45:15.132732",
        "created": "2013-11-27T17:45:15.084483",
        "resource_action": "vm_attach",
        "resource_ref": "http://yada.com",
        "impersonation_allowed": true,
        "verification_ref": "http://iad-int-api.cloudkeep.io:9311/v1/12345/verifications/b02f668e-8ccf-46a9-ba47-ec03b2233e4f",
        "is_verified": true,
        "resource_type": "image"
    },
    {
        "status": "ACTIVE",
        "updated": "2013-11-28T17:45:15.132732",
        "created": "2013-11-28T17:45:15.084483",
        "resource_action": "vm_attach",
        "resource_ref": "http://yada2.com",
        "impersonation_allowed": true,
        "verification_ref": "http://iad-int-api.cloudkeep.io:9311/v1/12345/verifications/b02f668e-8ccf-46a9-ba47-ec03b2233222",
        "is_verified": false,
        "resource_type": "image"
    }
  ]
}
```

The retrieved list of verifications is ordered by oldest to newest `created` date.

## Verifications Summary
> https://.../v1/{tenant_id}/verifications/

| Method | Description |
|--------|-------------|
| GET    | Returns a list of all verifications for a customer. |
| POST   | Starts the process of verifying a resource. This call will return immediately with a 202 OK and a link to the detail verification object (see below). |

> https://.../v1/{tenant_id}/verifications/{verification_uuid}

| Method | Description |
|--------|-------------|
| GET    | Returns the detailed verification data including the result of the verification. |
| PUT    | **Not allowed**. |
| DELETE | Cancels a verification. |

## Error Responses

| Action | Error Code | Notes |
|--------|------------|-------|
| POST verification with invalid data | 400 | Can include schema violations such as the verification's resource-type not supported |
| GET verification that doesn't exist | 404 | The supplied UUID doesn't match a verification in the data store |
| DELETE verification that doesn't exist | 404 | The supplied UUID doesn't match a verification in the data store |
