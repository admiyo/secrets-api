
# Examples

The following are example combinations, inspired by [this page](http://stackoverflow.com/questions/11946920/http-content-negotiation-compression-use-base64-with-accept-encoding-content-en).

The tables in this section are focused on the content-types and content-encodings of the various REST verb flows, even though each flow might have a different way to specify these values (either via http header settings or JSON request field). The reason for this approach is that while each flow has a different means to specify the mime-type and encoding, the values set for them must still be consistent with valid mime-type or encoding selections.

## One-Step UTF-8/ASCII Secret Create/Retrieve

| Action | content-type | content-encoding | Result |
|--------|-------------|-------|-----|
| POST secrets | `payload_content_type` = `text/plain` | `payload_content_encoding` not needed | Supplied `payload` is encrypted |
| GET secrets (meta) | `Accept: application/json` | Not required/ignored | JSON metadata, with `Content-Types` set to `'default':'text/plain'` |
| GET secrets | `Accept: text/plain` | Not required/ignored | Previous `payload` is decrypted |


## One-Step Binary Secret Create/Retrieve

| Action | content-type | content-encoding | Result |
|--------|-------------|-------|-----|
| POST secrets | `payload_content_type` = `application/octet-stream` | `payload_content_encoding` = `base64` | Supplied `payload` is converted from base64 to binary, then encrypted |
| GET secrets (meta) | `Accept: application/json` | Not required/ignored | JSON metadata, with `Content-Types` set to `'default':'application/octet-stream'` |
| GET secrets (decrypted) | `Accept: application/octet-stream` | Not specified | Previous `payload` is decrypted and returned as raw binary, _even if the PUT provided the data in `base64`_.  |

## Two-Step Binary Secret Create/Retrieve

| Action | content-type | content-encoding | Result |
|--------|-------------|-------|-----|
| POST secrets | `payload_content_type` optionally specified | `payload_content_encoding` optionally specified | Only metadata is created. If the `payload_content_type` or `payload_content_encoding` fields were provided, they are not used or saved with the metadata. The PUT request (next) will determine the secret's content type |
| PUT secrets (option #1 - as base64) | `Content-Type: application/octet-stream` | `Content-Encoding: base64` | Supplied request body is _converted from base64 to binary_, then encrypted |
| PUT secrets (option #2 - as binary) | `Content-Type: application/octet-stream` | Not specified | Supplied request body is encrypted as is |
| GET secrets (meta) | `Accept: application/json` | Not required/ignored | JSON metadata, with `Content-Types` set to `'default':'application/octet-stream'` |
| GET secrets (decrypted) | `Accept: application/octet-stream` | Not specified | Previous request is decrypted and returned as raw binary, _even if the PUT provided the data in `base64`_. |


## Two-Step Plain-Text Secret Create/Retrieve

| Action | content-type | content-encoding | Result |
|--------|-------------|-------|-----|
| POST secrets | `payload_content_type` optionally specified | `payload_content_encoding` optionally specified | Only metadata is created. If the `payload_content_type` or `payload_content_encoding` fields were provided, they are not used or saved with the metadata. The PUT request (next) will determine the secret's content format |
| PUT secrets | `Content-Type: text/plain` | Not required/ignored | Supplied request body is encrypted as is |
| GET secrets (meta) | `Accept: application/json` | Not required/ignored | JSON metadata, with `Content-Types` set to `'default':'text/plain'` |
| GET secrets (decrypted) | `Accept: text/plain` | Not specified | Previous request is decrypted and returned as utf-8 text |
