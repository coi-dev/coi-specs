# COI Server Spec v1.0

## Status & Discussion
Status: DRAFT

## Introduction
COI - Chat Over IMAP - provides a basis for realizing modern messaging services on top of the existing email infrastructure. COI leverages the existing open and federated email infrastructure to provide an open chat functionality accessible to everyone with an email address.

COI is designed to work (fallback mode) with existing, non COI-compliant IMAP4rev1 servers and IMAP4rev1 clients.  A COI-compliant server provides additional, optional features and optimizations recommended to both improve the COI user experience and make server handling of COI messages more efficient.  This specification describes the functionality of a COI-compliant server.

COI is not limited to IMAP; it is abstracted to allow any message accessing protocol (e.g. JMAP) to handle the data.  However, non COI-compliant server behavior was modeled on a base IMAP4rev1 server, as it is the most ubiquitous message retrieval protocol at the time it was designed, so this version of the specification focuses exclusively on IMAP behavior.  A future version of the spec may be extended to other protocols.

## Conventions Used in This Document
In examples, "C:" indicates lines sent by a client that is connected to a server. "S:" indicates lines sent by the server to the client.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

## COI Actors
In a typical COI setup the following actors are involved:

* A service/protocol ("server") for retrieving the message data, e.g. IMAP.  The server may or may not be COI-compatible. A COI compatible server is called "COI server" or just "server" in this specification.
* An SMTP service for sending messages. Sending services are also called Mail Transfer Agents (MTAs).
* Clients that access the server.

## Required Standards
### IMAP
An IMAP COI server MUST implement the following standards:

* IMAP 4rev1 [[RFC 3501](https://tools.ietf.org/html/rfc3501)]
A subset of IMAP METADATA Extension [RFC 5464]; REQUIRES support for private annotations, see below
* IMAP WebPush Extension (draft to be published soon)

#### Scope of METADATA Support
A COI server MUST support the following subset of the IMAP METADATA Extension [[RFC 5464](https://tools.ietf.org/html/rfc5464)] standard:

* `SETMETADATA` calls with the paths and value restrictions defined in this document as well as in the IMAP WebPush Extension v1.0.
* `GETMETADATA` calls with the paths defined in this document as well as in the IMAP WebPush Extension v1.0, this included support for the `DEPTH` parameter.

A COI server MAY announce the METADATA capability to clients. A COI server MAY reject any paths that are not defined in this document or in IMAP WebPush Extension v1.0.

## Recommended Standards
### IMAP
The following standards are RECOMMENDED to be supported by a COI server:

* IMAP4 IDLE Command [[RFC 2177](https://tools.ietf.org/html/rfc2177)] and IMAP NOTIFY Extension [[RFC 5465](https://www.rfc-archive.org/getrfc.php?rfc=5465)]

  *Used to poll for new messages*
* IMAP4 Binary Content Extension [[RFC 3516](https://tools.ietf.org/html/rfc3516)]

  *Used to reduce traffic between client and server*
* IMAP Extension for Referencing the Last SEARCH Result [[RFC 5182](https://tools.ietf.org/html/rfc5182)] and IMAP ESORT Extension [[RFC 5267](https://tools.ietf.org/html/rfc5267)]

  *Improves efficiency of client sync *
* IMAP4 Quick Flag Changes Resynchronization (CONDSTORE) and Quick Mailbox Resynchronization (QRESYNC) [[RFC 7162](https://tools.ietf.org/html/rfc7162)]

  *Improves efficiency of client sync*

## COI Capability
As COI implements functionality layered on top of the standard messaging protocols, it is necessary to be able to query a server to determine whether that server is COI-compatible.

### IMAP
A COI IMAP server will advertise it's COI compatibility via the IMAP `CAPABILITY` response.

For a COI server, the following capabilities MUST be listed in the response:

* COI
* WEBPUSH


*Example:*

```
S: * OK IMAP Server
C: a CAPABILITY
S: * CAPABILITY IMAP4rev1
S: a OK
C: b LOGIN foo bar
S: b OK Authentication successful.
C: c CAPABILITY
S: * CAPABILITY IMAP4rev1 COI WEBPUSH
S: c OK
```

## Getting COI Namespace Configuration
Servers can be customized to use different namespaces.

### IMAP
Servers MUST provide the configuration using a `GETMETADATA` call with using the `/private/vendor/vendor.dovecot/coi/config` path prefix.

The following settings can be available under this config path, details are explained in the subsequent sections:

* `mailbox-root` whose value contains the root mailbox name for COI-related messages. Servers MAY allow to set a user-individual `mailbox-root`. In the following the value of this setting is called *\<MAILBOX-ROOT>*.
* `enabled` which defines if COI is enabled for this user.
* `message-filter` which defines the filtering of COI messages.
Only mailbox-root MUST exist for any user of a COI server.

*Example for reading configuration:*

```
C: a GETMETADATA (DEPTH 1) "" /private/vendor/vendor.dovecot/coi/config
S: * METADATA "" ("/private/vendor/vendor.dovecot/coi/config/mailbox-root" "COI")
S: * METADATA "" ("/private/vendor/vendor.dovecot/coi/config/enabled" "yes")
S: * METADATA "" ("/private/vendor/vendor.dovecot/coi/config/message-filter" "active")
S: a OK GETMETADATA complete
```

*Example for unsuccessfully setting the `mailbox-root`:*

```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/coi/config/mailbox-root myroot)
S: a NO [CANNOT] unable to set /private/vendor/vendor.dovecot/coi/config/mailbox-root
```

## Configure COI Usage
As COI implements functionality layered on top of the standard messaging protocols, it is necessary to enable COI behavior only in accounts that will use that behavior.

### IMAP
Clients indicate their desire for a mail account to begin supporting server-side COI optimizations by issuing the `SETMETADATA` command using the `/private/vendor/vendor.dovecot/coi/config/enabled` path.

Allowed values are:

* `yes` for enabling COI. It is RECOMMENDED that this value is treated case-sensitive, i.e. only lower-case value are valid.
* `NIL` for disabling COI. This removes the setting.

Other values SHOULD result into a NO response. Any other value than `yes` SHOULD be interpreted as COI being disabled.

When reading the state, clients should check if the path `/private/vendor/vendor.dovecot/coi/config/enabled` is present and if the value for it is `yes`. Only when both statements are true, COI is enabled for the user.

When the COI is enabled by the client and before returning a tagged OK response, the server MUST do the following:

* Check for the existence and create the following folders if they do not yet exist:
  * *\<MAILBOX-ROOT>/Contacts* for retrieving configuration values from clients. By default this is the *COI/Contacts* folder. The folder MAY be marked as subscribed on behalf of the user.
  * *\<MAILBOX-ROOT>/Chats* for storing  chat messages. By default this is the *COI/Chats* folder. The folder SHOULD be marked as subscribed on behalf the user. Compare the message filtering section below for details. It is RECOMMENDED that this folder cannot be renamed by clients - or only renamed (moved) to the trash folder.
* Evaluates the `/private/vendor/vendor.dovecot/coi/config/message-filter` configuration and initiates filtering accordingly, compare the filtering configuration section below. When the configuration is not specified, the default value `none` is assumed.

When COI is disabled, the server SHOULD stop filtering COI messages. The server MAY stop adding the `$Chat` keyword to detected COI messages. The server MAY remove webpush registrations that apply to COI messages only. 

*Example for enabling COI:*

```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/coi/config/enabled yes) 
S: a OK SETMETADATA complete
```

*Example for disabling COI:*

```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/coi/config/enabled NIL) 
S: a OK SETMETADATA complete
```

*Example for invalid value:*
```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/coi/config/enabled true) 
S: a NO [CANNOT] invalid value for /private/vendor/vendor.dovecot/coi/config/enabled: "true"
```

### Indirectly Disabling and Enabling COI
It is RECOMMENDED that servers treat the deletion of the *\<MAILBOX-ROOT>/Chats* folder as an indirect request to disable COI for the user. This allows users to disable COI without needing to use a COI compliant client. A deletion can either happen either directly with the IMAP `DELETE` command or indirectly by a `RENAME` operation that moves the *Chats* folder under a `\Trash` special-use or the default trash folder. It is RECOMMENDED to not allow other `RENAME` operations, such attempts are RECOMMENDED to be answered with NO.

It is RECOMMENDED that servers treat the creation of the *\<MAILBOX-ROOT>/Chats* folder as an indirect request to enable COI for the user. The creation can be done either with a `CREATE` or with a `RENAME` call.

If the server indirectly enables or disables COI, it MUST also update the corresponding METADATA setting under `/private/vendor/vendor.dovecot/coi/config/enabled`.

## Configure Message Filtering
For non-COI compatible servers, messages are required to be organized into the COI folder namespace via COI-client behavior.  However, on COI compliant servers this can be done by the server. Once a user enables COI for an account, messages can be filtered automatically by the server at delivery, preventing the overhead of manual message manipulation on the client-side.

COI messages are detected by having at least one these characteristics:

* The `Chat-Version` header is present.
* The `Message-ID` header starts with `chat$`.
* The first message-ID in the `References` header starts with `chat$`.

When a `message-filter` value of `active` or `seen` is present, a server MUST check for the existence and create the following folder if they do not yet exist:

* *\<MAILBOX-ROOT>/Chats* for moving COI messages, this defaults to *COI/Chats*. The folder SHOULD be marked as subscribed by the user.

### IMAP
Clients configure the filtering for new incoming COI messages by using `SETMETADATA` command using the `/private/vendor/vendor.dovecot/coi/config/message-filter` path.

The following filter modes MUST be supported using this meta data:

* `none`: messages are not separated at all. This is the default value.
* `active`: messages are moved to *\<MAILBOX-ROOT>/Chats* folder. By default this is *COI/Chats*.
* `seen`: messages stay in the *INBOX* folder until they are flagged with `\Seen` flag. When the COI server receives a `\Seen` flag for a COI message, then it will move the message like in the active mode.

Servers MAY respond with NO when a different value is encountered. It is RECOMMENDED that values are treated case-sensitive, i.e. only lower-case values are valid.

When a COI message has been detected, the server MUST mark the message with the *$Chat* keyword.

If the server supports Sieve or other filtering mechanisms, COI message filter rules are RECOMMENDED be applied before any user-defined filtering rules are evaluated. Global filtering rules MAY be executed before applying COI message filter rules. It is RECOMMENDED to understand the Sieve keep rule so that COI messages will stay within *\<MAILBOX-ROOT>/Chats* when the `message-filter` is set to `active`.

COI message filter rules MUST be applied before the message is actually stored in the target mailbox, so that an existing IMAP `IDLE` command is not triggered for the *INBOX*, when a COI message is received and the `message-filter` rule is set to `active`.

*Example for enabling active filtering:*

```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/coi/config/message-filter active)
S: a OK SETMETADATA complete
```

*Example for invalid filtering value:*
```
C: a SETMETADATA "" (/private/vendor/vendor.dovecot/coi/config/message-filter inactive)
S: a NO [CANNOT] invalid value for /private/vendor/vendor.dovecot/coi/config/message-filter: inactive
```
