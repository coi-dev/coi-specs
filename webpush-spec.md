# IMAP WebPush Extension v1.0
## Status & Discussion
Status: DRAFT

## Introduction
Historically, IMAP clients have used `IDLE` or `NOTIFY` to be notified about new messages. These methods require a persistent connection, which may be difficult for mobile clients. These mobile clients often require an external service that listens for incoming messages on behalf of the user; such services may require the user's credentials, which poses a security threat.

This document describes a method to deliver push message to clients without storing user credentials and without persistent connections. 

## Conventions Used in This Document
In examples, "C:" indicates lines sent by a client that is connected to a server. "S:" indicates lines sent by the server to the client.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

## IMAP WebPush Actors
In a typical setup the following actors are involved:

* An IMAP WebPush-compatible client. Also called just "WebPush client" or "client" or "MUA" (Mail User Agent).
* An IMAP server that supports WebPush as described in this specification. Such a compatible server is called "WebPush server" or just "server" in this specification.
* A client-specific WebPush compliant web push service that allows clients to register push resources and that accepts push requests from the WebPush server.

The following figure informs about the main flows in the push architecture.

```
    +-------+           +--------------+       +--------------+
    |  MUA  |           | Push Service |       | IMAP WebPush |
    +-------+           +--------------+       |   Server     |
        |                      |               +--------------+
        |      Subscribe       |                      |
        |--------------------->|                      |
        |       Monitor        |                      |
        |<====================>|                      |
        |                      |                      |
        |          Distribute Push Resource           |
        |-------------------------------------------->|
        |                      |                      |
        |                      |   Validation Message |  
        |   Validation Message |<---------------------|
        |<---------------------|                      |
        |                      |                      |
        |          Validate Push Resource             |
        |-------------------------------------------->|
        :                      :                      :
        |                      |     Push Message     |
        |    Push Message      |<---------------------|
        |<---------------------|                      |
        |                      |                      |
```
                      Figure 1: IMAP WebPush Architecture

## Required Standards
The IMAP WebPush Extension is based on the following standards:
* Generic Event Delivery Using HTTP Push [[RFC 8030](https://tools.ietf.org/html/rfc8030)]
* Voluntary Application Server Identification (VAPID) for Web Push [[RFC 8292](https://tools.ietf.org/html/rfc8292)]
* Message Encryption for Web Push [[RFC 8291](https://tools.ietf.org/html/rfc8292)]

### IMAP
These IMAP extensions are REQUIRED:
* IMAP4rev1 [[RFC 3501](https://tools.ietf.org/html/rfc3501)]
* A subset of IMAP METADATA Extension [[RFC 5464](https://tools.ietf.org/html/rfc5464)]; REQUIRES support for private annotations (see below)

These IMAP extensions are RECOMMENDED:
* IMAP4 Non-synchronizing Literals [[RFC 7888](https://tools.ietf.org/html/rfc5464)]

#### Scope of METADATA Support
A WebPush server MUST support the following subset of the IMAP METADATA Extension [[RFC 5464](https://tools.ietf.org/html/rfc5464)] standard:

* `SETMETADATA` calls with the paths and value restrictions defined in this document.
* `GETMETADATA` calls with the paths defined in this document and support for the `DEPTH` parameter.

A WebPush server is not required to announce the `METADATA` capability to clients. A WebPush server MAY reject any paths that are not defined in this document.

In this specification draft, a vendor specific prefix is used as part of the `METADATA` path (*vendor.dovecot*). It is planned that this path will change in the future (e.g. to *webpush*) once this standard is officially recognized.

## Capability
The server MUST advertise the `WEBPUSH` capability to announce compliance with this specification.  This capability SHOULD NOT be advertised in non-authenticated state.

*Example:*
```
S: * OK IMAP Server
C: a CAPABILITY
S: * CAPABILITY IMAP4rev1
S: a OK
C: b LOGIN foo bar
S: b OK [CAPABILITY IMAP4rev1 WEBPUSH] Authentication successful.
```

## VAPID Key
Push services need to verify if a push request is coming from a valid source. This is done by signing the push request headers with the VAPID key (see [[RFC 8292](https://tools.ietf.org/html/rfc8292)] for details).

Clients can determine the VAPID key by using the `GETMETADATA` command. The VAPID key MAY be user-specific or shared among all clients connected to an IMAP server.  It is located at */private/vendor/vendor.dovecot/webpush/vapid*.

The server MUST return the VAPID key in the base 64-encoded form of the PEM encoded key, for details compare [[RFC 1421](https://tools.ietf.org/html/rfc1421)]. 

*Example:*
```
C: a GETMETADATA "" /private/vendor/vendor.dovecot/webpush/vapid
S: * METADATA "" (/private/vendor/vendor.dovecot/webpush/vapid {133}
S: -----BEGIN PUBLIC KEY-----
S: MDkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDIgACYHfTQ0biATut1VhK/AW2KmZespz+
S: DEQ1yH3nvbayCuY=
S: -----END PUBLIC KEY-----)
S: a OK Getmetadata completed (0.130 + 0.000 + 0.129 secs).
```
## Subscriptions
WebPush subscriptions are stored in the */private/vendor/vendor.dovecot/webpush/subscriptions/* path. Clients can register and manage subscriptions.

### Subscribe
A WebPush server MUST support subscribing to push events.

Clients can subscribe to push notifications by storing a subscription with a `SETMETADATA` call that specifies the push details as a server annotation. The metadata entry name MUST start with */private/vendor/vendor.dovecot/webpush/subscriptions/* followed by the client-generated, unique ID of the push registration.

The push registration ID MUST follow the following constraints:

* The push registration ID MUST only contain ASCII alphanumeric or minus or underline characters:  `[0-9A-Za-z_-]`.
* The push registration ID MUST be at least 10 and at most 36 characters long.

Accepted contents for the data is a JSON-serialized object with the following entries:

* `client`: the name of the client (string, max-length is 128 octets)
* `device`: the name of the device (string, max-length is 128 octets)
* `msgtype`: string, the type of messages for which notifications are requested. Possible values are:
  * `"chat"`: for chat messages, for the purpose of this documentation a chat email message is marked with the $Chat keyword . Compare for example the [COI Server Specification v1.0](coi-server-spec.md) for how to detect chat messages.
  * `"mail"`: for standard mail messages, excluding chat email messages.
  * `"all"`: for both standard mail and chat messages.
* resource: object, the push resource as specified by [[RFC 8030](https://tools.ietf.org/html/rfc8030)] and [[RFC 8292](https://tools.ietf.org/html/rfc8292)] with the children `endpoint` and `keys`.
  * `endpoint` specifies the URL for sending push requests (string, max length is 512 octets).
  * `keys` specifies the encryption keys:
    * `p256dh`: contains the client-generated key for encryption message payload. The key used Elliptic Curve Diffie-Hellman (ECDH) on the P-256 curve as defined by [[RFC 8291](https://tools.ietf.org/html/rfc8291)] (string, max-length is 256 octets).
    * `auth`: 16-octets long client-generated symmetric secret used for the key generation as defined by [[RFC 8291 section 3.2](https://tools.ietf.org/html/rfc8291#section-3.2)].

It is RECOMMENDED to treat `msgtype` values case sensitive and to not allow any other values than specified above.

The server SHOULD ignore unknown key value pairs in the provided JSON structure, meaning they will not be saved.

When the server has received a new subscription, the server MUST add the created JSON element, which is a  [[ISO 8601](https://tools.ietf.org/html/rfc8291#section-3.2)] formatted date time with the server's date and time of the subscription creation. The server MAY create the `validated` JSON element, which is false by default until the subscription has been validated, compare the "Validate New Subscriptions" section.

A server MUST allow several clients to be subscribed at the same time. The server is, however, RECOMMENDED to limit the number of concurrent subscriptions. If the maximum number is hit, it MUST answer a subscription request with `NO`. The client then needs to remove an existing subscription first.



*Example for subscribing for receiving push notifications when Literal+ is supported [[RFC 7888](https://tools.ietf.org/html/rfc7888)]:*
```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/webpush/subscriptions/31754ee7-d3ee-4226-b112-6895ed26fcf8 {370+}
{
    "client": "My awesome WebPush App",
    "device": "My Favorite Smartphone",
    "msgtype": "chat",
    "resource":
    {
        "endpoint": "https://my-push-service.com/some-kind-of-unique-id-1234/v2/",
        "keys": {
            "p256dh": "BNcRdreALRFXTkOOUHK1E...P7DkM=",
            "auth": "tBHItJI5svbpez7KI4CCXg=="
        }
    }
}
)
S: a OK SETMETADATA complete
```

### Validate New Subscriptions
Every actor within a web push subscription has an interest to validate push subscriptions to ensure that every actor is legitimate.

This is done by using the following steps:

* The server sends a validation push message request.
* This allows the push service to validate that a newly created push resource is actively being used. The push service also forwards the validation message to the client.
* This allows the client to validate that the server is actually fully webpush compliant. The client decrypts the validation message and uses that to validate the original subscription with the server.
* This last step allows the server to validate that the whole webpush chain is compliant.

#### Sending a Validation Push Message
When a new subscription is registered, the server initiates the validation by sending a push notification with a simple `Application/JSON` payload that contains the single `validation` element. This element contains a text with a maximum size of 1kB that allows the server to validate the subscription later. How the server creates the validation is explicitly not  specified. A server MAY sign a text and timestamp and use the signature for validation, for example.

Additionally, the request towards the push service requires the additional HTTP header `WebPush-Validation` without any value.

*Example payload:*
```
{
  "validation": "abc$123$..."
}
```
#### Validating A Push Subscription
When the client receives a validation message, it will validate the previous subscription by calling `SETMETADATA` with the same path as the original subscription followed by */validate*. The value is the received value of the validation JSON element of the validation message. 

When this `SETMETADATA` call is received within a certain timeframe - RECOMMENDED is 5 minutes - of sending the validation push message and with the correct value for the given subscription ID, the server MUST respond with OK. When more than the timeframe has passed, or the validate text is invalid or the subscription ID does not match, the server MUST respond with NO.

When no validation is received within the timeframe after creation, the server SHOULD discard the subscription and the client needs to regenerate it.

*Example code for validating a subscription within the required timeframe:*
```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/webpush/subscriptions/31754ee7-d3ee-4226-b112-6895ed26fcf8/validate "abc$123$...")
S: a OK SETMETADATA complete
```
*Example code for validating a subscription outside of the required timeframe or with an invalid value:*
```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/webpush/subscriptions/31754ee7-d3ee-4226-b112-6895ed26fcf8/validate "ZZZ000...")
S: a NO [CANNOT] invalid validation
```

### Unsubscribe
To unsubscribe from push notification set the previously push annotation to `NIL`.

*Example for unsubscribing from push notifications: *
```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/webpush/subscriptions/31754ee7-d3ee-4226-b112-6895ed26fcf8 NIL)
S: a OK SETMETADATA complete
```

### List Existing Subscriptions
To list existing subscriptions, call `GETMETADATA` with a `DEPTH` of 1 to the path */private/vendor/vendor.dovecot/webpush/subscriptions*.

Note that the server will also list the `created` key using a [[RFC 3339](https://tools.ietf.org/html/rfc3339)] format timestamp value and the `validated` key, which is either `true` or `false`.

*Example for listing existing push notification subscriptions:*
```
C: a GETMETADATA (DEPTH 1) "" (/private/vendor/vendor.dovecot/webpush/subscriptions)
S: * METADATA "" (/private/vendor/vendor.dovecot/webpush/subscriptions/123-123-123-123-123123 {380}
S: {"created":"2019-10-16T16:14:17+00:00","validated":true,"client":"OX COI Messenger","device":"mydevicename","msgtype":"chat","resource":{"endpoint":"https://example.com:443/push/send/123-123-123-123","keys":{"p256dh":"keydata=","auth":"Authdata="}}})
a OK Getmetadata completed (0.002 + 0.000 + 0.001 secs).
S: a OK GETMETADATA complete
```

## Push Notification Payload
An incoming message that matches subscription requirements and that is not a disposition notification message MUST trigger a notification of the corresponding endpoint of that subscription.

### Payload Data Format
The server SHOULD include the following fields in an JSON object that is the payload. Note that if string fields have to be truncated, they MUST NOT truncated in the middle of an UTF8 sequence or the middle of any escape sequence such as  `\uXXXX`  or `\"`.

* `from-email`: the lowercase UTF8 email address extracted from the `From` header, at least the first 100 bytes as a JSON string, if truncated followed by an ellipsis `U+2026`.
* `from-name`: the name extracted from the `From` header, at least the first 100 bytes as a JSON string, if truncated followed by an ellipsis `U+2026`. If there is no name given, then this field SHOULD be omitted.
* `subject`:  as specified in the `Subject` header,  at least the first 100 bytes as a JSON string, if truncated followed by an ellipsis `U+2026`. If no or only an empty Subject is defined, then this field SHOULD be omitted.
* `content`: if the final encrypted payload stays under 4kB, the full body will be included.
* `content-type`: if `content` is specified, then `content-type` SHOULD be specified, at the least the first 200 bytes as a JSON string, if truncated followed by an ellipsis `U+2026`. If `content-type` is `text/plain` this field SHOULD be omitted.
* `content-encoding`: if `content` is specified, then `content-encoding` SHOULD be specified, unless it is `8bit`.
* `date`: the date of the message in [[RFC 3339](https://tools.ietf.org/html/rfc3339)]-compatible timestamp, if no `Date` header is present, this field SHOULD be omitted.
* `msg-id`: the ID of the incoming message (ASCII text), at least the first 100 bytes as a JSON string, if truncated followed by an ellipsis `U+2026`.
* `uid`: the UID of the incoming message (32 bit integer)
* `uidvalidity`: the UID validity of the folder (32 bit integer)
* `folder`: the mailbox of the incoming message, if longer than 200 bytes as a JSON string this field SHOULD be omitted.

When the server is compatible with the [COI Client Specification v1.0](coi-client-spec.md), then it SHOULD additionally provide the following fields:
* `group-id`: the ID of the group extracted from the Message-ID or the References header, compare https://github.com/coi-dev/coi-specs/blob/master/coi-client-spec.md#group-messages. This field SHOULD be only present when the message is actually part of a group discussion. Max-length is 128 octets for this field.

A push service is not required to support more than 4096 octets of payload body (see [Section 7.2 of [RFC8030](https://tools.ietf.org/html/rfc8030#section-7.2)]). Absent header (86 octets), padding (minimum 1 octet), and expansion for AEAD_AES_128_GCM (16 octets), this equates to, at most, 3993 octets of plaintext.

*Example:*
```
{
    "from-email":  "sender@domain.com",
    "from-name":  "Alice",
    "subject": "Chat: Hello",
    "content": "--------------MIME-BOUNDARY------------\nContent-Type: application/octet-stream; name=\"encrypted.asc\"[...]",
    "content-encoding": "multipart/encryptedl;protocol=\"application/pgp-encrypted\";boundary=\"--------------MIME-BOUNDARY------------\""
    "date": "2019-12-31T23:20:50.52Z",
    "msg-id": "chat$2923892892@domain.com",
    "uid": 2398293,
    "uidvalidity": 123287,
    "folder": "COI/Chats"
}
```
### Push Message Encryption
Encrypting a push message uses Elliptic Curve Diffie-Hellman (ECDH) on the P-256 curve to establish a shared secret and a symmetric secret for authentication.

A user agent generates an ECDH key pair and authentication secret that it associates with each subscription it creates. The ECDH public key and the authentication secret are sent to the application server with other details of the push subscription.

When sending a message, an webpush server generates an ECDH key pair and a random salt. The ECDH public key is encoded into the `keyid` parameter of the encrypted content coding header, and the salt is encoded into the `salt` parameter of that same header (see [Section 2.1 of [RFC8188](https://tools.ietf.org/html/rfc8188#section-2.1)]). The ECDH key pair MUST be discarded after encrypting the message.

The content of the push message is encrypted or decrypted using a content encryption key and nonce. These values are derived by taking the `keyid` and `salt` as input to the process described in [Section 3 of [RFC 8291](https://tools.ietf.org/html/rfc8291#section-3)].

The processes and details are described in detail in [[RFC 8291](https://tools.ietf.org/html/rfc8291#section-3)].

### Header Signing
A WebPush server MUST authorize their push requests by using the vapid HTTP authentication scheme as defined by [[RFC 8292](https://tools.ietf.org/html/rfc8292)]. This scheme consists of JSON Web Token (JWT) that contains several claims and that is signed.

The "t" parameter of the Authorization header field contains a JWT; the "k" parameter includes the base64-url-encoded key that signed that token.

*Example for VAPID header:*
```
Authorization: vapid
      t=eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3
        B1c2guZXhhbXBsZS5uZXQiLCJleHAiOjE0NTM1MjM3NjgsInN1YiI6Im1ha
        Wx0bzpwdXNoQGV4YW1wbGUuY29tIn0.i3CYb7t4xfxCDquptFOepC9GAu_H
        LGkMlMuCGSK2rpiUfnK9ojFwDXb1JrErtmysazNjjvW2L9OkSSHzvoD1oA,
      k=BA1Hxzyi1RUM1b5wjxsn7nGxAszw2u61m164i3MrAIxHF6YK5h4SDYic-dR
        uU_RCPCfA5aq9ojSwk5Y2EmClBPs
```
### Push Message Example
```
POST /p/JzLQ3raZJfFBR0aqvOMsLrt54w4rJUsV HTTP/1.1
   Host: push.example.net
   TTL: 30
   Content-Length: 136
   Content-Encoding: aes128gcm
   Authorization: vapid
      t=eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3
        B1c2guZXhhbXBsZS5uZXQiLCJleHAiOjE0NTM1MjM3NjgsInN1YiI6Im1ha
        Wx0bzpwdXNoQGV4YW1wbGUuY29tIn0.i3CYb7t4xfxCDquptFOepC9GAu_H
        LGkMlMuCGSK2rpiUfnK9ojFwDXb1JrErtmysazNjjvW2L9OkSSHzvoD1oA,
      k=BA1Hxzyi1RUM1b5wjxsn7nGxAszw2u61m164i3MrAIxHF6YK5h4SDYic-dR
        uU_RCPCfA5aq9ojSwk5Y2EmClBPs
   { encrypted push message }
```

### Push Service Response Codes
Push services MUST deliver one of the following response codes for push requests:

* `201 Created`. The request to send a push message was received and accepted.
* `429 Too many requests`. Meaning your application server has reached a rate limit with a push service. The push service should include a `Retry-After` header to indicate how long before another request can be made.
* `400 Invalid request`. This generally means one of your headers is invalid or improperly formatted.
* `404 Not Found`. This is an indication that the subscription is expired and can't be used. In this case you should delete the PushSubscription and wait for the client to resubscribe the user.
* `410 Gone`. The subscription is no longer valid and should be removed from application server. This can be reproduced by unsubscribing a push subscription.
* `413 Payload size too large`. The minimum size payload a push service must support is 4096 bytes (or 4kB).

## IANA Considerations
The `WEBPUSH` capability has to be registered with IANA.

## Security Considerations
The WebPush-capable IMAP server requires to be able to access the Internet, which poses additional risks. Proxies are commended to be used.

Malicious web push services could be harming the IMAP server with the following means:
* Delay responses to push requests
* Do not respond at all

A WebPush-capable IMAP server MAY unsubscribe push subscriptions that are having such bad-behaving push service implementations. Clients need to check subscriptions and re-subscribe them if required.

Malicious clients could create many subscriptions that are invalid. The validation flow catches those problems, IMAP servers SHOULD introduce throttling of subscriptions to counter such behavior. If a client tries to register several invalid subscriptions, the server is RECOMMENDED to lock out that specific client or throttle subscriptions for the affected user.