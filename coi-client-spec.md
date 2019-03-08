# Status & Discussion
*Status: DRAFT*

Following topics need further discussion:
* Hashing algorithm - for now we assume SHA3-256 for hashing email addresses, this might be changed in the future.
* End-to-end encryption when the communication partners are on COI-enabled servers and use COI-compliant clients. [Message Layer Security](https://datatracker.ietf.org/wg/mls/about/) may provide a basis for that in the future.
* There are various todos spread throughout this document.

# Table of Contents

* [Status & Discussion](#status--discussion)
* [Introduction](#introduction)
* [Conventions Used in This Document](#conventions-used-in-this-document)
* [COI Actors](#coi-actors)
* [Checking for Server Side COI Support](#checking-for-server-side-coi-support)
* [Message Format](#message-format)
  * [Message Format Design Considerations](#message-format-design-considerations)
  * [Goals](#goals)
  * [Backwards Compatibility](#backwards-compatibility)
  * [COI-specific Message Header Fields](#coi-specific-message-header-fields)
  * [COI Base Formats](#coi-base-formats)
* [Messages](#messages)
  * [Text Messages](#text-messages)
  * [Binary Messages](#binary-messages)
  * [Preview Message Parts](#preview-message-parts)
  * [Edit and Delete Messages](#edit-and-delete-messages)
  * [Read Receipts / Disposition Notification Reports](#read-receipts--disposition-notification-reports)
    * [Requesting Read Receipts](#requesting-read-receipts)
    * [Sending Read Receipts](#sending-read-receipts)
  * [Contact Storage Messages](#contact-storage-messages)
    * [Contact Storage Message Headers](#contact-storage-message-headers)
    * [Contact Storage Message Body](#contact-storage-message-body)
    * [Contact Storage Message Example](#contact-storage-message-example)
    * [Encrypting Contacts](#encrypting-contacts)
    * [Blocking Contacts](#blocking-contacts)
    * [Merging Contact Storage Messages](#merging-contact-storage-messages)
  * [User Profile Messages](#user-profile-messages)
    * [User Profile Storage Message](#user-profile-storage-message)
    * [User Profile Information Message Part](#user-profile-information-message-part)
    * [Requesting Profile Information](#requesting-profile-information)
    *  [Processing Received Profiles](#processing-received-profiles)
  * [Group Messages](#group-messages)
    * [Sending a Chat Message to a Group](#sending-a-chat-message-to-a-group)
    * [Replying to a Group Message](#replying-to-a-group-message)
    * [Group Storage Message](#group-storage-message)
    * [Group Information Message Part](#group-information-message-part)
    * [Leaving a Group: Group Participant Left Message](#leaving-a-group-group-participant-left-message)
    * [Converting a 1:1 to a Group Conversation](#converting-a-11-to-a-group-conversation)
    * [Group Message Encryption](#group-message-encryption)
    * [Request Group Information](#request-group-information)
    * [Group Messaging and Mailing Lists](#group-messaging-and-mailing-lists)
    * [Mentions in Group Messages](#mentions-in-group-messages)
    * [Changing Group Definitions and Processing Changes](#changing-group-definitions-and-processing-changes)
      * [Processing Group Name Changes](#processing-group-name-changes)
      * [Processing Group Participants Changes](#processing-group-participants-changes)
      * [Processing Group Description and Avatar Changes](#processing-group-description-and-avatar-changes)
      * [Processing Group Participant Changes](#processing-group-participant-changes)
      * [Processing Group Changes](#processing-group-changes)
  * [Location Messages](#location-messages)
  * [Pinned Messages](#pinned-messages)
  * [Poll Messages](#poll-messages)
  * [Extension Messages](#extension-messages)
* [Message Encryption](#message-encryption)
* [Separate COI and Mail Messages](#separate-coi-and-mail-messages)
* [Configuring Separation on COI Servers](#configuring-separation-on-coi-servers)
* [Manual Message Seperation](#manual-message-seperation)
* [Folders](#folders)
* [Security Considerations](#security-considerations)
* [IANA Considerations](#iana-considerations)

# Introduction
COI - Chat Over IMAP - provides a basis for realizing modern messaging services on top of the existing email infrastructure. COI leverages the existing open and federated email infrastructure to provide an open chat functionality accessible to everyone with an email address.

COI will work with standard IMAP servers and IMAP clients and will work better with COI-aware clients and servers. Initially, most email servers will not provide support for COI, so we need ways how to realize the most important functions on top of the existing infrastructure.

# Conventions Used in This Document
In examples, "C:" indicates lines sent by a client that is connected to a server. "S:" indicates lines sent by the server to the client.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

# COI Actors
In a typical COI setup the following actors are involved:

* A COI-compatible client, i.e. your app. Also called just "COI client" or "client".
* An service for retrieving the message data, currently we use IMAP for that, but COI is not tied to IMAP and could be made available through JMAP too, for example. The IMAP server may or may not be COI-compatible. The IMAP service MUST be compatible with IMAP v4rev1 as defined in [RFC 3501](https://tools.ietf.org/html/rfc3501), which is the case for almost any IMAP service in production.
* An SMTP service for sending messages. Sending services are also called Mail Transfer Agents (MTAs).
* Other clients, which may or may not be COI-compatible. Clients are also called Mail User Agents (MUAs).
Ideally, your client will be able to communicate with both COI-compatible and normal email clients, however this decision is your choice. You can also choose to only process COI-compliant messages in your app, which will reduce efforts but it might also reduce the potential audience for your users.

A typical COI communication sequence includes the following steps:

1. Connect to your IMAP service and check for COI compliance - if it is supported, enable it for the currently signed in user.
2. Send specifically formatted mail messages over SMTP. 
3. Listen for incoming messages, differentiate between chat, normal and blocked messages and move them to the respective folders.
4. Show chat messages to the end user.

# Checking for Server Side COI Support 
A COI client should check for COI support from the server first. 

After signing into the IMAP service with your user credentials, call `CAPABILITY`. When the server returns the "COI" capability you will know that this is supported. You will find out configuration values and activate COI support for your user by calling `ENABLE COI`.

Please refer to the forthcoming COI Server Specification for details.

# Message Format
Any chat message MUST be compliant with the [RFC 5322](https://tools.ietf.org/html/rfc5322) definition. If binary data is transmitted, the message additionally needs to conform to the MIME message standard ([RFC 2045](https://tools.ietf.org/html/rfc2045), [RFC 2046](https://tools.ietf.org/html/rfc2046), and [RFC 2049](https://tools.ietf.org/html/rfc2049)).

# Message Format Design Considerations
## Goals
With the COI specification we try to achieve the following goals:

* **Backwards Compatibility**
Always have something to show for existing email clients. Each message SHOULD contain 
at least one text/plain part,
additionally optionally a text/html part, or
a binary part like image/jpeg. See below for more details.
* **Open Standards**
Use open and existing standards when possible, e.g. 
  * [h-card microformat](http://microformats.org/wiki/h-card) for contact information
  * Disposition Notification Reports for read receipts
  * application/json attachments for structured data
* **Secure Messaging**
Allow clients to use any encryption mechanism like GPG or Autocrypt.org. 
Optional direct IMAP 2 IMAP communication

* **Modern User Experience** 
Provide support for allowing a user experience that most people expect, e.g. 
  * Easy cross device contact sync
  * Allow to block users
  * Edit & delete sent messages
  * Useful interactions like polls
  * Push (with COI enabled backend)
  * Typing notifications (when both parties use a COI-enabled servers)
## Backwards Compatibility
COI messages are designed to be backwards compatible with existing email clients. While some COI functionalities will not work on such legacy clients, users of non-COI-compliant clients should still be informed about it. Therefore, each message MUST have at least one part that explains the content of this message. These part MIME types are RECOMMENDED:
  * text/plain for simple text
  * text/html, or
  * a common binary format like for example image/jpeg.

For the same reason, the number of messages SHOULD be limited - ideally, instead of sending automated messages, a COI client only sends out user-generated messages and possibly enriches those.

If possible, commonly understood MIME parts such as image/jpeg or text/vcard are used, so that again non-COI compliant clients can at least view nested content parts. For some messages, it is better not to include attachments directly but to embedd relevant data within the HTML message parts instead. In this way non-technical users are not confused by attachments they cannot understand.



# COI-specific Message Header Fields
COI-aware email clients and COI IMAP servers MUST send messages conforming to the header descriptions below:

* A COI message MUST contain the `Chat-Version` header field with a value of "1.0". Note that further parameters MAY follow which will be separated by semicolons, e.g. `Chat-Version: 1.0;name=value`.
* A COI message MUST only contain one address in the "From" field.
* A COI message MUST contain a `Date` header field with a value adhering to [RFC 5322](https://tools.ietf.org/html/rfc5322#section-3.3).
* A COI message MUST contain a `Message-ID` header field in the following format: "coi$" + domain-unique ID + "@" + mail domain, e.g. `<coi$KJKJK2312321312KJ@mydomain.org>`.   
* A COI message MUST contain `MIME-Version: 1.0` in the header.
* A COI message SHOULD contain a `References` header field with the message-IDs of the initial message and the last message only. Note that this recommendations is contrary to the RFC 5322 recommendation, which results in all message-ID are all parent messages being referenced. However, in a chat conversation there will be way more messages than in a typical email thread, so it would be impractical to reference all previous messages.
* A COI message SHOULD contain a `In-Reply-To` header when answering to a previous message. This header field contains the message-ID in of the replied message.
* The `To` header contains the recipient(s) of the message. 
* The `From` header field of a COI message MUST only contain one sender.
* A COI message SHOULD NOT have a `Sender` header field.
* A COI message MAY contain a `Disposition-Notification-To` header field. If it is present the field content MUST be the same as the `From` header field. This is for requesting read reports.
* A binary COI message MUST contain the `Content-Transfer-Encoding` header field.
* A binary COI message SHOULD contain the `Content-Disposition: inline` header. It SHOULD also contain the filename and size parameters, for example `Content-Disposition: inline;filename="earth.jpg";size=2048`.
* The `Chat-Content` header MAY be used in message headers and message part headers to identify the contents of a specific part. Possible values include:
  * *ignore* for parts that should not be shown to end users by a COI compliant client.
  * *status* for a part that contains the user status.
  * *ammend* and *collapse* for edited and deleted messages.
  * *preview* for parts that contain a preview for a large message such as a video.
  * *contact* for a part that contains a full VCARD contact for syncing purposes.
  * *profile* for a part that contains information about the sending contact embedded in the HTML.
  * *profilerequest* for requesting contact information.
  * *group*  for a message that contains group information for cross device syncing.
  * *groupprofile* for a part that contains information about a group embedded in the HTML.
  * *groupprofilerequest* for requesting group information.
  * *groupleft* for a request to leave a group conversation.
  * *poll* for sending a poll.
  * *pollvote* for voting in a poll.
  * *pollclose* for closing a poll.
  * *client specific value* for any extension message parts that are client specific.
* Depending on the message contents or its encryption, additional headers MAY be set.
* A COI client MAY set the `Subject` header field.  

# COI Base Formats
A COI message will use one of these formats as specified in the `Content-Type` header field:

* *text/plain*: for normal, user generated text messages without any additional data
* *multipart/alternative*: for user generated messages in which some parts may be applicable for a COI client while the `text/plain` or `text/html` part is applicable for both non-COI-compliant clients and COI clients.
* *multipart/mixed*: for creating more complex messages, e.g. a contact storage message that contains a `text/vcard` part of the sender as an attachment.
* *multipart/signed*: a multipart message that is additionally signed.

# Messages
## Text Messages
User generated text messages SHOULD be delivered in the `text/plain` mime format with a UTF-8 encoding when possible. This is defined with following header field: `Content-Type: text/plain; charset=UTF-8; format="flowed"`.
Note that incoming messages from existing email clients MAY be delivered in other formats but SHOULD still be shown. Any existing signatures and quote blocks SHOULD be either stripped away or folded out of view. A client MAY show an option to view the full message contents.

*Example for a simple text message:*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$S2571BC.2878&8@sample.com> 
Content-Type: text/plain; charset=UTF-8; format="flowed" 
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
MIME-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com> 
 
Hello World
My dear friend, this is a message to you!
```

When a client supports formatting messages e.g. with markdown, it should convert that message to HTML code. The client MAY additionally provide a `text/markdown` part in this case. Additionally, a client can mark parts with `Chat-Content: ignore`, to hide certain parts from COI compliant clients. 

Note that clients will choosewhich part they want to show based on client-specific considerations. Security-minded COI clients may prefer `text/plain` parts wheras others may prefer `text/markdown` or `text/html` parts for a richer experience.

*Example for a formatted text message:*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$22938.8238702@sample.com> 
Content-Type: multipart/alternative;
 boundary=unique-boundary-1
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com>
MIME-Version: 1.0

--unique-boundary-1
Content-Type: text/plain; charset=UTF-8

hello COI world!


--unique-boundary-1
Content-Type: multipart/mixed;
 boundary=unique-boundary-2

--unique-boundary-2
Content-Type: text/html; charset=UTF-8

<p>hello <b>COI</b> world!</p>

--unique-boundary-2
Content-Type: text/html; charset=UTF-8
Chat-Content: ignore

<p><i>This is a chat message 
- consider using <a href="https://myawesomecoiapp.com">my awesome COI app</a> for best experience!</i></p>

--unique-boundary-2--

--unique-boundary-1
Content-Type: text/markdown; charset=UTF-8

hello **COI** world!

--unique-boundary-1--
```


An initial message SHOULD contain contact information about the sender, please compare the Contact Messages section below for details.

# Binary Messages

Typical examples for binary messages are
* photos: `image/webp`, `image/jpeg` and `image/png` SHOULD be supported by a COI-compatible client. `image/gif` is RECOMMENDED to be supported. 
* audio-recordings: `audio/webm` SHOULD be supported by a COI-compatible client. `audio/mp4` and `audio/aac` is  RECOMMENDED to be supported.
* videos: `video/webm` SHOULD be supported by a COI-compatible client. `video/mp4` is RECOMMENDED to be supported.

In a chat environment, a single message typically only contains one binary message, but it MAY also contain several attachments at once.

A binary message MUST contain the following additional fields:
* `Content-Transfer-Encoding`: the encoding of the message, base64 MUST be supported, compare [RFC 4648](https://tools.ietf.org/html/rfc4648) for details.
* `Content-Disposition`: the disposition of the binary data. It SHOULD have the value "inline". Additionally the parameters "filename" and "size" (in octets=bytes) SHOULD be defined. Compare [RFC 2183](https://tools.ietf.org/html/rfc2183) for details.
* Clients SHOULD support messages with multiple files. In this case, the corresponding `multipart/mixed` messages need to be supported. If the COI client supports encryption, additionally `multipart/encrypted` messages MUST be supported.

*Example 1: A message containing a single image.*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$434571BC.8070702@sample.com> 
Content-Type: image/jpeg
Content-Transfer-Encoding: base64
Content-Disposition: inline;filename=earth.jpg;size=2048
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com>
MIME-Version: 1.0 

TWFuIGlzIGRpc3Rpbmd1aXNoZWQsIG5vdCBvbmx5IGJ5IGhpcyByZWFzb24sIGJ1dCBieSB0aGlz
IHNpbmd1bGFyIHBhc3Npb24gZnJvbSBvdGhlciBhbmltYWxzLCB3aGljaCBpcyBhIGx1c3Qgb2Yg
dGhlIG1pbmQsIHRoYXQgYnkgYSBwZXJzZXZlcmFuY2Ugb2YgZGVsaWdodCBpbiB0aGUgY29udGlu
dWVkIGFuZCBpbmRlZmF0aWdhYmxlIGdlbmVyYXRpb24gb2Yga25vd2xlZGdlLCBleGNlZWRzIHRo
ZSBzaG9ydCB2ZWhlbWVuY2Ugb2YgYW55IGNhcm5hbCBwbGVhc3VyZS4=
```

*Example 2: A multipart message containing one audio recording and one image.*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$123AB223.8238702@sample.com> 
Content-Type: multipart/mixed;
 boundary=unique-boundary-1
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com>
MIME-Version: 1.0

--unique-boundary-1
Content-Type: audio/webm
Content-Transfer-Encoding: base64
Content-Disposition: inline

... base64-encoded WebM audio data goes here ...

--unique-boundary-1
Content-Type: image/jpeg
Content-Transfer-Encoding: base64
Content-Disposition: inline;
 filename="venus.jpg";size=4096
 
... base64-encoded JPEG image data goes here ...

--unique-boundary-1--
```

### Preview Message Parts

For a large binary message such as a video or a high resolution photo, a COI-compatible client MAY add a preview message part. This allows clients with unstable or rate-limited network connections to show the message without needing to download the full message contents first.

A COI client MUST be able to process such a preview part and not show two individual parts. It MAY ignore such message parts to be compliant.

A preview message sets the `Chat-Content` header field to "preview" in the respective part:

```
Chat-Content: preview
```

A preview message part typically contains one lower resolution image. 

*Example: A video with an image as a preview:*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$123AB223.8238702@sample.com> 
Content-Type: multipart/mixed;
 boundary=unique-boundary-1
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com>
MIME-Version: 1.0

--unique-boundary-1
Content-Type: image/jpeg
Content-Transfer-Encoding: base64
Content-Disposition: inline
Chat-Content: preview

... base64-encoded jpeg data goes here ...

--unique-boundary-1
Content-Type: video/webm
Content-Transfer-Encoding: base64
Content-Disposition: inline;
 filename="bigbuckbunny.webm";size=4096
 
... base64-encoded webm video data goes here ...

--unique-boundary-1--
```

Clients should first download the envelope data of very large messages to detect preview parts. When such a part is available and network connectivity is limited, it is RECOMMENDED that only the preview part is downloaded until the user requests the full content.

## Edit and Delete Messages

To allow users to change already sent messages, a follow up edit message can be send. A COI-compatible client SHOULD only show the latest edited message, but it MAY make the history available. An edit message MAY be empty, this means that the original message should be shown as deleted. A COI client SHOULD visualize the edited state of a message.

An edit message MUST contain set the `Chat-Content` header field to "ammend" and the `In-Reply-To` header field to the message-ID of the edited, for example:

```
Chat-Content: ammend
In-Reply-To: <coi$434571BC.8070702@sample.com>
```
If a message should be deleted, then the COI client MUST set the `Chat-Content` header field to "collapsed":

```
Chat-Content: collapse; 
In-Reply-To: <coi$434571BC.8070702@sample.com>
```

A message can be edited and/or deleted several times and COI client SHOULD only show the latest version. An edit and delete message MAY contain a read receipt request, which SHOULD be processed normally. To not create a sense of false security, it is RECOMMENDED that clients show the history of edited and delete messages. A deleted message is RECOMMENDED to be shown as "this message has been deleted".

A COI client MUST ensure that the referenced message is coming from the same sender as the original message.

Multiple edits SHOULD reference the last edited message, not the original message.

If a the referenced message is not found, e.g. because it is deleted, then the only new message SHOULD be shown.

If a client decides not to support edit and delete message, it just shows all messages in their chronological order.

*Example 1: A normal message is being sent with a typo:*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$S2571BC.287888@sample.com> 
References: <coi$434571BC.89A707D2@sample.com> 
Content-Type: text/plain; charset=UTF-8 
Chat-Version: 1.0
MIME-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com> 
 
My dear friend, this sdkalsk.
```

*Example 2: The above message is being edited with a follow up message:*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$2131232.AKJSD@sample.com> 
References: <coi$434571BC.89A707D2@sample.com> 
Content-Type: text/plain; charset=UTF-8 
Chat-Version: 1.0
Chat-Content: edit;
In-Reply-To: <coi$S2571BC.287888@sample.com
MIME-Version: 1.0
 
My dear friend, this is a message to you!
```

*Example 3: The above message is being deleted with a follow up message:*

```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$KJ23928L.112222312&8@sample.com> 
References: <coi$434571BC.89A707D2@sample.com> 
Content-Type: text/plain; charset=UTF-8 
Chat-Version: 1.0
Chat-Content: collapsed;
In-Reply-To: <coi$2131232.AKJSD@sample.com>
MIME-Version: 1.0  
 
(original message marked as deleted)
```

## Read Receipts / Disposition Notification Reports

COI-compatible clients can request and send read notifications by following the disposition notification standards defined [RFC 8098](https://tools.ietf.org/html/rfc8098) and [RFC 3503](https://tools.ietf.org/html/rfc3503). 

### Requesting Read Receipts
You can can request read receipts by adding the `Disposition-Notification-To` header field with a value of your email address, for example:
```
Disposition-Notification-To: Me Myself <me@example.com>
```

### Sending Read Receipts
If the user approves sending read requests, a COI-compatible client SHOULD send our corresponding reports when receiving a chat message that:
* contains the `Disposition-Notification-To` header field, and
this value is not equals the user's email address, and
* the message has not set the $MDNSent keyword, and
* the message has been shown to the user.

A COI-compatible client SHOULD then sent out a message disposition notification using the following format:

```
From: Alice <you@recipientdomain.com>
To: Bob <me@example.com>
Subject: Chat: Read receipt
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$122B223.823202@sample.com> 
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
Content-Type: multipart/report; 
 report-type=disposition-notification;
 boundary="RAA14128.773615765/recipientdomain.com"
Content-Disposition: notification
MIME-Version: 1.0

--RAA14128.773615765/recipientdomain.com
Content-Type: message/disposition-notification

Reporting-UA: joes-pc.cs.recipientdomain.com; Foomail 97.1
Original-Recipient: rfc822;you@recipientdomain.com
Final-Recipient: rfc822;you@recipientdomain.com
Original-Message-ID: <coi$199509192301.23456@sample.com>
Disposition: manual-action/MDN-sent-manually; displayed

--RAA14128.773615765/recipientdomain.com--
```

To reduce traffic, a COI-compliant client SHOULD NOT include the original message like suggested by [RFC 8098](https://tools.ietf.org/html/rfc8098). 

After sending the read receipt, the client MUST set the `$MDNSent` keyword for the original message.

*TODO describe how to do that.*

## Contact Storage Messages

To synchronize contacts across devices, a client MAY store contacts as individual messages into the *COI/CONTACTS* folder. Note that on a COI-compatible IMAP server this folder is generated automatically, compare the COI server specification for details.  Also, the *COI* folder prefix may be different depending on the result of the `ENABLE COI` IMAP command.

To keep the user in control, a COI client SHOULD ask the user for permission, before storing contacts on the server.

In order to allow even non-COI-compliant clients to view contact details, COI clients SHOULD use an extended version of the `h-card` microformat embedded in a `text/html` message (or message part).

Clients SHOULD encrypt the message body to protect the contact data.

### Contact Storage Message Headers
To efficiently process contact messages on the server side, some data must be stored in the headers, specifically the `From`, `Chat-Content`, `COI-Token` (`COI-TokenIn` and `COI-TokenOut`) and `COI-Blocked` header fields that are described below.

* `From`: can contain the email address of the signed-in user.
* `COI-From-Hash`: to allow servers to locate contact storage messages, the `COI-From-Hash` contains the SHA3-256 hash of the normalized form of the email address of the contact. Normalized form means removing quotes, using UTF-8 and lowercasing the email address. If the contact has several email addresses, one `COI-From-Hash` header is specified for each email address. 
* `To`: Optional the logged-in user's primary address for this contact. 
* `Subject`: "Contact" or some other informative text.
* `Date`: Timestamp when the contact was last intentionally modified (which can be different from received-time, which is when it was uploaded to server)
* `COI-Token` (`COI-TokenIn` and `COI-TokenOut`): COI compatible servers use tokens to allow direct IMAP server to IMAP server message transfers. There can be several COI-Token header fields for a single contact. The token structure is defined in the COI server specification. Clients SHOULD NOT use or change the token information.
* `Chat-Content`: set to "contact" for contacts or to "profile" for profile information of the currently signed in user.
* `Content-Type`: is either `text/html` or `multipart/alternative` or `multipart/encrypted`.

### Contact Storage Message Body


A contact storage message SHOULD be a `text/html` message (or message part) with the following mandatory fields:

* `p-name` the name of the user
* `u-email` the email address of the user (can be repeated for several email addresses)

The following optional fields are COI-specific or structured in a COI-specific way:
* `p-category`: COI clients use the following predefined categories values. Several categories can be separated by a COMMA.
  * *friend* for friends
  * *family* for family members
  * *work* for work colleagues
  * *partner* for the partner
  * other value for client or user specific categorization, e.g. *sports*.
* `p-status`: the status of the user e.g. "gone fishing"
* `p-notification`: notification settings for this users with a SEMICOLON separated list of setting key-value pairs, each pair separated by an equals sign:
  * `enabled`: is deemed enabled unless set to "false" or "no".
  * `types`: a COMMA separated list of client specific type values like *sound* or *banner*.
  * `sounds`: a COMMA separated list of client specific sound values. RECOMMENDED values include the same values as the *categories* field.
  * `filters`:  criteria when a notification should be shown. When missing and is_enabled is true, it is RECOMMENDED that a notification is shown for any new user generated message. Possible values are
    * *message* for any user generated message excluding read receipts, contact updates, etc,  
    * *mention* for messages that contain a mention of this user, or a 
    * client specific value

Compare the [h-card microformat](http://microformats.org/wiki/h-card) specification for details.



### Contact Storage Message Example
The following example shows a contact storage message for a contact with the email addresses *john.doe@coi-dev.org* and *john.doe@example.com*:


```
From: Alice <alice@example.com>
COI-From-Hash: 74b3c446c189580045a942b90b7461128bc334b3b67e9f63c0c06189248c9bb0
COI-From-Hash: 834d1654133366cef4d81e81a7099f413b1b85b9fc9dd62140491303dec56532
Subject: Contact
Date: Mon, 4 Dec 2019 17:51:37 +0100 
Message-ID: <coi$22323i89.AB29702@sample.com> 
Content-Type: text/html; charset=UTF-8
Chat-Version: 1.0
MIME-Version: 1.0
Chat-Content: contact

<html>
<body style="font-family: sans">
<p class="h-card" style="white-space: nowrap">
<img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
<span class="p-name">John Doe</span><br>
Nickname: <span class="p-nickname">Johnny</span><br>
Email: <span class="u-email">john.doe@coi-dev.org</span><br>
Email: <span class="u-email">john.doe@example.com</span><br>
<span class="p-status">working from home...</span>
<span class="p-category" style="display: none">friend, work</span>
<time class="dt-bday" style="display: none">--12-09</time>
<span class="p-notification" style="display: none">enabled=true;
 types=sound,banner;sounds=friend;filters=message,mention</span>
</p>
</body>
</html>
```

### Encrypting Contacts

To protect the privacy of contact, the complete message body of a contact storage message SHOULD be encrypted. It is RECOMMENDED that clients use the user's GPG/PGP key for encrypting the message body. The `Content-Type` header of the message is then set to `multipart/encrypted`.

### Blocking Contacts
To mark a contact as blocked a COI client MUST set the `COI-Blocked` header field of the corresponding contact storage message. The actual content of this header field is ignored but is RECOMMENDED to be set to "yes". When a contact is blocked, a COI client SHOULD move any incoming messages to the *Blocked* folder, unless the IMAP server is COI compliant, in which case the server does that. 

Before blocking a contact, the client SHOULD check the last received message of that contact for a DKIM header. In case there is no DKIM header, the client SHOULD warn the user about the possibility, that the sender address might be spoofed. This assumes that the server checks the DKIM header and rejects invalid headers.

### Merging Contact Storage Messages
COI clients need to be able to merge mails if they find duplicates as identified by having a same `COI-From-Hash` header.

 * When finding a token, find it from all mails, even if they are duplicates
 * Contact is updated by `APPEND`-ing a new one and expunging the old one.
 * If mails are found that have duplicate `COI-From-Hash` headers, those mails should be merged.
 * Merging is done by merging all of those mails' headers.
   * All `COI-TokenIn` headers are preserved, except expired tokens can be dropped.
   * `COI-TokenOut` headers preserve the latest ones (newest create timestamp) for each unique "hash", while dropping others
   * All other headers are preserved. In case the of conflicts, use the newer mail's fields (newer Date header, or in case they equal then newer IMAP `UID`)


## User Profile Messages
### User Profile Storage Message

A user profile storage message is structured like a contact storage message with the following differences:
* There can be several message parts each containing a different profile description. This allows clients to differentiate the information they are sharing with a contact based on the category of a contact. As an example, users might want to share their birthday with their friends, but not with other contacts.
* Each message part MUST contain the `Chat-Content: profile` header, with the optional `categories` parameter defining the COMMA separated group-categories, for which the profile is applicable, for example `Chat-Content: profile; categories=friend, family`.


*Example for a user profile message with a profiles for three different audiences:*
```
From: Alice <alice@example.com>
Subject: Profile
Date: Mon, 4 Dec 2019 17:51:37 +0100 
Message-ID: <coi$KJK2342KJ.K232989KJ@sample.com> 
Content-Type: multipart/mixed; 
 boundary="SDKJK2323.23123/example.com"
Chat-Version: 1.0
MIME-Version: 1.0
Chat-Content: profile

--SDKJK2323.23123/example.com
Content-Type: text/html; charset=UTF-8
Chat-Content: profile; categories=friend,family

<html>
<body style="font-family: sans">
<p class="h-card" style="white-space: nowrap">
<img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
<span class="p-name">Alice Angel</span><br>
Nickname: <span class="p-nickname">Al</span><br>
Email: <span class="u-email">alice@example.com</span><br>
Email: <span class="u-email">angel@work-example.com</span><br>
<span class="p-status">weekend here I come...</span>
<time class="dt-bday" style="display: none">1998-12-24</time>
</p>
</body>
</html>

--SDKJK2323.23123/example.com
Content-Type: text/html; charset=UTF-8
Chat-Content: profile; categories=work

<html>
<body style="font-family: sans">
<p class="h-card" style="white-space: nowrap">
<img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
<span class="p-name">Alice Angel</span><br>
Email: <span class="u-email">angel@work-example.com</span><br>
<span class="p-status">working from home...</span>
</p>
</body>
</html>


--SDKJK2323.23123/example.com
Content-Type: text/html; charset=UTF-8
Chat-Content: profile

<html>
<body style="font-family: sans">
<p class="h-card" style="white-space: nowrap">
<img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
<span class="p-name">Alice Angel</span><br>
Email: <span class="u-email">alice@example.com</span><br>
<time class="dt-bday" style="display: none">--12-24</time>
</p>
</body>
</html>

--SDKJK2323.23123/example.com--

```

To protect the user's information, clients SHOULD encrypt the complete body of the profile storage message. It is RECOMMENDED that clients use the user's GPG/PGP key for encrypting the message body. The `Content-Type` of the message is then set to `multipart/encyrpted`.


### User Profile Information Message Part

COI clients MAY use contact information messages to transmit profile data such as their image. This helps in setting up a good user experience. It is RECOMMENDED that clients send out updated profile data with new user generated messages only. However, a COI client MAY send this updated profile to all active contacts. What is deemed active and if a COI client differentiates between contacts is a client specific decision. It is RECOMMENDED that all contacts from which chat messages have been received in the last 30 days are deemed active.

Profile information is embedded in the `text/html` part of a `multipart/alternative` message. The part MUST be marked with a `Chat-Content: profile` header.


To reduce unnecessary traffic, it is RECOMMENDED that a COI client keeps track which contacts have received the profile information message already. COI clients MAY use a client specific header field in a contact storage message, e.g. `com_example_myawesesomeclient_TimeProfiletSent: 2019-03-19T07:22Z`.



*Example for a message with a profile part:*
```
From: Alice <alice@example.com> 
To: bob@recipientdomain.com
Subject: Chat: Hello my ...
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$238788.AB29702@sample.com> 
Content-Type: multipart/alternative; 
 boundary="SDKJK2233.23278827/sample.com"
Chat-Version: 1.0
MIME-Version: 1.0

--SDKJK2233.23278827/sample.com
Content-Type: text/plain; charset=UTF-8; format="flowed"
Content-Disposition: inline

Hello my friend, this is my first COI message!

--SDKJK2233.23278827/sample.com
Content-Type: text/html; charset=UTF-8
Content-Disposition: inline
Chat-Content: profile

<html>
<body style="font-family: sans">
<p>Hello my friend, this is my first COI message!</p>
<hr>
<p class="h-card" style="white-space: nowrap">
<img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
<span class="p-name">Alice Angel</span><br>
Email: <span class="u-email">alice@example.com</span><br>
Birthday: <span class="dt-bday">--04-30</span><br>
<span class="x-coi-status">working from home...</span>
</p>
</body>
</html>

--SDKJK2233.23278827/sample.com--
```



### Requesting Profile Information

When a COI client has lost information about a contact and cannot restore the information neither from the COI/CONTACTS IMAP folder nor from the message history, it MAY request the contact information by setting the Chat-Content header field to "profilerequest". It is RECOMMENDED to send the request along with a normal user generated message. The receiving COI client SHOULD send the contact information to the requesting party. It is RECOMMENDED to ask the recipient for approval before sending the contact information.

*Example for a profile request message:*
```
From: Bob <bob@example.com> 
To: Alice <alice@example.com> 
Subject: Chat 
Date: Mon, 4 Dec 2019 15:23:13 +0100 
Message-ID: <coi$S2323BC.2878&8@sample.com> 
Content-Type: text/plain; charset=UTF-8; format="flowed"
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
Chat-Content: profilerequest
MIME-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com> 

Hi Alice, can you please advise me?
```

### Processing Received Profiles
*TODO describe parsing and storing contact informations through generating the respective HTML, checking the COI-From-Hash header of existing messages first, then encrypting the message body*


## Group Messages

Sending messages to a group is simple in principle: just send message to several participants that are defined in the `To` field. 

Each group has an assigned ID. The group ID MUST only contain alphanumeric , minus and underline characters:  `[0-9A-Za-z_-]+`. A group ID MUST be case-sensitive and MUST contain at least 12 characters.

To identify groups in replies from non-COI-compliant clients, the group ID is embedded into the message ID in the format `coi$group.<groupd ID>.<domain unique value>@<domain>`, e.g.

```
Message-ID: <coi$group.4321dcba1234.ABSKDJK23223.293892839@example.com>
```

Groups are identified by parsing this `Message-ID` or - in the case of replies from non-COI-compliant clients - the first message-ID in the `References` header field.

The group-ID is also acting as an identifier, i.e. COI clients MUST allow to have the same participants in different groups. As an example, users can start a topic-specific discussion by creating a new group, even if the group only contains two participants.

It is RECOMMENDED that the group can be managed like email, i.e. any user can add or remove anyone from the list of recipients by changing either `To` or `CC`. Any group member SHOULD be able to change the group name, description and avatar.

To enable group avatars, allow participants to leave a group, support to manage groups, etc., COI clients use additional, specifically formatted message parts that are described below. 

### Sending a Chat Message to a Group
A COI client sends a message to a group by adding several contacts to the `To` header field. If a group-name exists, a COI client SHOULD specify the group name with the `Subject` header field and the group participants in the `To` header field. To not accidentally spread user-specific nicknames for contacts, COI clients SHOULD either use names authorized by the participants  or use the email addresses without names in the `To` header field.

*Example for a message to a group called "My Crew" and a group ID of `4321dcba1234`:*

```
From: Me Myself <me@example.com> 
To: alice@example.com; bob@example.com
Subject: My Crew
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$group.4321dcba1234.S2571BC.2878&8@sample.com> 
Content-Type: text/plain; charset=UTF-8 
References: <coi$group.4321dcba1234.434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
MIME-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com> 
 
Hi gang, hope you're doing fine!
```

### Replying to a Group Message

When replying to a group message, COI clients MUST reply to all group participants by default, similarly to the "reply all" function of most email clients. COI clients MAY offer switching to a 1:1 conversation based on a group participant's message or from the group participant's list.


### Group Storage Message

To sync group information across devices, a client MAY use group storage messages. 


Group storage messages are identified by a `Chat-Content: group` header. The `Content-Type` is set to `multipart/mixed` or `multipart/encrypted`. The message contains one `text/html` part. 

A group storage message contains the following data in the parent element `h-group`:

* `p-Id`: The ID of the group.
* `p-name`: The name of the group.
* `p-description`: the optional description of the group
* `u-photo`: the optional group avatar
* `p-notification`: The notification settings for this group, which has the same key value pairs structure as for contact storage messages.
* `e-participants`: A list of participants, each one either identified by a `h-card`. An optional  hash of the email address acts as a pointer to a corresponding contact storage message. The hash is a SHA3-256 hash of the normalized email address; compare the definition of the `COI-From-Hash` header of contact storage messages. 


Group storage messages SHOULD be stored in the *COI/CONTACTS* folder, unless the `ENABLE COI` IMAP command returns a differently configured folder prefix. Compare the COI server spec for details.

*Example for a group storage message:*
```
From: Me Myself <me@example.com> 
Subject: Group
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$group.1234abcd4321.238788.AB29702@example.com> 
Content-Type: text/html; charset=UTF-8
Chat-Version: 1.0
MIME-Version: 1.0
Chat-Content: group

<html>
<body>
 <p class="h-group">
<img class="u-photo" style="float: left" alt="" src="data:image/png;base64,
iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAC4HpUWHRSYXcgcHJvZmlsZSB0eXBl
IGV4aWYAAHja7ZddsuMoDIXfWcUswZIQEsvB/FT1Dmb5c8Bc5yb3dldNTz/MQ0wFsCwEnE84Seh/
/xjhL1yU5QhRzVNO6cAVc8xc0PHjuq6WjrjqdfF+hPsne7gfMEyCVq7b1Ld/gV0fAyxu+/lsD1Z3
HN+B9oOPgDJnnrNtP9+BhC877fuQ97gSP21nf8SunXw4v95HgxhNYRQO3IXkQD0HsmAFkqWglVUn
nhZCn8RRq8TvtQt390W8u/ei3VG2XZ6lCEfaDulFo20n/V67pdDnFdFj5qcHJvcUX7Qbo/kY/dpd
iQlKpbA39bGV1YPjCSllDUsoho+ib6tkFMcWK0RvoHmi1ECZGGoPitSo0KC+2koVS4zc2dAyV5Zl
czHOXBeUOAsNNuBpASxYKqgJzHyvhda8ec1XyTFzI3gyIRhhxJcSvjP+TrkDjTFTl+jwWyusi2cC
YhmT3KzhBSA0tqa69F0lfMqb4xNYAUFdMjs2WI7zCnEqPXJLFmeBnx4xHFe6k7UdABJhbsVikNGR
jkSilOgwZiOCjg4+BStniXyCAKlyozDARnASjJ3n3BhjtHxZ+TLj1QIQKkkMaHCAACtGRf5YdORQ
UdEYVDWpqWvWkiTFpCklS/MdVUwsmloyM7dsxcWjqyc3d89eMmfBK0xzyhay55xLwaQFoQtGF3iU
cvIpZzz1TKedfuazVKRPjVVrqla95loaN2k4/i01C81bbqVTRyr12LWnbt177mUg14aMOHSkYcNH
HuWmtqk+U6MXcr+mRpvaJBaXnz2owWz2EYLm60QnMxDjSCBukwASmiezwylGnuQmsyMzDoUyqJFO
OI0mMRCMnVgH3ewe5H7JLWj8V9z4Z+TCRPcnyIWJbpP7yu0baq2sbxRZgOYpnJoeMvBig0P3wl7m
d9Jvt+G/BngHegd6B3oHegd6B3oH+v8EGvjxgL+a4R8RTpDYz0xx+AAAAAZiS0dEAP8ASgAATip0
5QAAAAlwSFlzAAALEwAACxMBAJqcGAAAAAd0SU1FB+MDBgoSOo2gyjcAAAGmSURBVFjD7da/axRR
EAfwj5qAgshJkHQ2KVKIsUwhMaS1Eosk/gPBSrGwif+BWKvYp4k/qtxhpUUSkUNIEYSAIKY1drkj
akjOZg6WZd9mT4wQ2C9Ms/OdN9/3Zua9pUaNGjWKcQ/b2Mc6JnP+a2ihG9bCRI4zg00cYgePcKpq
8l7OdjGWSd5JcPoiruJnAWexioDtgsAeHoe/lfD30AzO84R/p4qA/UTwUvi7JQI6wWkm/If5ZKcL
BLQTwtoD9NCHxPfNKsGTUc+s8jbODVCCC/ic8+1huuoOxqLmS7ifSS4abfeIJuyLWMQKnuLKvxzT
idhtJ6xZMIYnA0UXw2Xcjlk+jx/4iDcxAVXQwByu41Kc0gZe4ksqaBhP8DvRYC8yghux+K0QO4WL
mU29LhnDZzhbNI6poC7uxMJzWMVBAe8Aa5gP7gJ+JdZ8izNZAQsJ4h5uYBTvSsYvb+8j5mbJxfYg
K2A5QbqLEWwNkLxvWxH7sOQUDIWAr/iUK8v3qPsrjP9Fg49H/Gz0SiPn/6bi89g7hkkrfQv+K4Yq
cGbr/7MaNY4TfwDbd9AYja7LyQAAAABJRU5ErkJggg==">
  Name: <span class="p-name">My Crew</span><br/>
  Description: <span class="p-description">This is just a fun place to hang out virtually</span><br/>
  <p class="p-id" style="display: none;">1234abcd4321</p>
  <span class="p-notification" style="display: none">enabled=true;types=banner;filters=mention</span>
  <hr/>
  <p>Participants</p>
  <div class="e-participants">
    <p class="h-card" style="white-space: nowrap">
	  <p class="p-hash" style="display: none;">5457683cbe3191ab696f4d0a1c71ff48105d733d406239bb45439969672ea51c</p>
      <img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
      <span class="p-name">Alice Angel</span><br>
      Email: <span class="u-email">alice@example.com</span><br>
      PGP: <span class="u-key">...base64 key data...</span><br/>
      Birthday: <span class="dt-bday">--04-30</span><br>
      <span class="x-coi-status">working from home...</span>
    </p>
    <p class="h-card">
      <img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
      <span class="p-name">John Doe</span><br>
      Email: <span class="u-email">john.doe@example.com</span><br>
      PGP: <span class="u-key">...base64 key data...</span><br/>
    </p>
    <p class="h-card">
      <img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
      <span class="p-name">Caroll</span><br>
      Email: <span class="u-email">carrol@example.com</span><br>
      PGP: <span class="u-key">...base64 key data...</span><br/>
    </p>
  </div>
 </p>
</body>
</html>
```

### Group Information Message Part

When creating a new group, COI clients SHOULD include the group description in the first user generated message. 

The group information is embedded in a `text/html` part that is additionally taggged with a `Chat-Content: groupinfo` header.

The `groupinfo` message part uses a similar data structure like the group storage message but uses a [microformat](http://microformats.org/wiki/microformats2) for rendering the information:
* `h-group` is the parent element for the group
  * `u-photo` (optional) group avatar image
  * `p-description` (optional) group description
  * `p-participants` contains the list of participants, each in an individual `h-card` element. Compare the contact information message for details about that format.

Depending on the message content, the message SHOULD either be `multipart/alternative` for text-based contents or `multipart/mixed` for binary contents or when additionally a `contactinfo` part is sent. Encrypted messages will use the `multipart/encrypted` content type.


*Example for a message to a group that contains a groupinfo part:*

```
From: Alice <alice@example.com> 
To: john.doe@example.com; carrol@example.com
Subject: My Crew
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$group.1234abcd4321.238788.AB29702@example.com> 
Content-Type: multipart/alternative; 
 boundary="A22090S.123213/example.com"
Chat-Version: 1.0
MIME-Version: 1.0

--A22090S.123213/example.com
Content-Type: text/plain; charset=UTF-8; format="flowed"
Content-Disposition: inline

Hello folks, let's chat!

--A22090S.123213/example.com
Content-Type: text/html; charset=UTF-8
Content-Disposition: inline
Chat-Content: groupinfo

<html>
<body>
 <p>Hello folks, let's chat!</p>
 <div style="display: none;">
 <hr/>
 <p class="h-group">
<img class="u-photo" style="float: left" alt="" src="data:image/png;base64,
iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAC4HpUWHRSYXcgcHJvZmlsZSB0eXBl
IGV4aWYAAHja7ZddsuMoDIXfWcUswZIQEsvB/FT1Dmb5c8Bc5yb3dldNTz/MQ0wFsCwEnE84Seh/
/xjhL1yU5QhRzVNO6cAVc8xc0PHjuq6WjrjqdfF+hPsne7gfMEyCVq7b1Ld/gV0fAyxu+/lsD1Z3
HN+B9oOPgDJnnrNtP9+BhC877fuQ97gSP21nf8SunXw4v95HgxhNYRQO3IXkQD0HsmAFkqWglVUn
nhZCn8RRq8TvtQt390W8u/ei3VG2XZ6lCEfaDulFo20n/V67pdDnFdFj5qcHJvcUX7Qbo/kY/dpd
iQlKpbA39bGV1YPjCSllDUsoho+ib6tkFMcWK0RvoHmi1ECZGGoPitSo0KC+2koVS4zc2dAyV5Zl
czHOXBeUOAsNNuBpASxYKqgJzHyvhda8ec1XyTFzI3gyIRhhxJcSvjP+TrkDjTFTl+jwWyusi2cC
YhmT3KzhBSA0tqa69F0lfMqb4xNYAUFdMjs2WI7zCnEqPXJLFmeBnx4xHFe6k7UdABJhbsVikNGR
jkSilOgwZiOCjg4+BStniXyCAKlyozDARnASjJ3n3BhjtHxZ+TLj1QIQKkkMaHCAACtGRf5YdORQ
UdEYVDWpqWvWkiTFpCklS/MdVUwsmloyM7dsxcWjqyc3d89eMmfBK0xzyhay55xLwaQFoQtGF3iU
cvIpZzz1TKedfuazVKRPjVVrqla95loaN2k4/i01C81bbqVTRyr12LWnbt177mUg14aMOHSkYcNH
HuWmtqk+U6MXcr+mRpvaJBaXnz2owWz2EYLm60QnMxDjSCBukwASmiezwylGnuQmsyMzDoUyqJFO
OI0mMRCMnVgH3ewe5H7JLWj8V9z4Z+TCRPcnyIWJbpP7yu0baq2sbxRZgOYpnJoeMvBig0P3wl7m
d9Jvt+G/BngHegd6B3oHegd6B3oH+v8EGvjxgL+a4R8RTpDYz0xx+AAAAAZiS0dEAP8ASgAATip0
5QAAAAlwSFlzAAALEwAACxMBAJqcGAAAAAd0SU1FB+MDBgoSOo2gyjcAAAGmSURBVFjD7da/axRR
EAfwj5qAgshJkHQ2KVKIsUwhMaS1Eosk/gPBSrGwif+BWKvYp4k/qtxhpUUSkUNIEYSAIKY1drkj
akjOZg6WZd9mT4wQ2C9Ms/OdN9/3Zua9pUaNGjWKcQ/b2Mc6JnP+a2ihG9bCRI4zg00cYgePcKpq
8l7OdjGWSd5JcPoiruJnAWexioDtgsAeHoe/lfD30AzO84R/p4qA/UTwUvi7JQI6wWkm/If5ZKcL
BLQTwtoD9NCHxPfNKsGTUc+s8jbODVCCC/ic8+1huuoOxqLmS7ifSS4abfeIJuyLWMQKnuLKvxzT
idhtJ6xZMIYnA0UXw2Xcjlk+jx/4iDcxAVXQwByu41Kc0gZe4ksqaBhP8DvRYC8yghux+K0QO4WL
mU29LhnDZzhbNI6poC7uxMJzWMVBAe8Aa5gP7gJ+JdZ8izNZAQsJ4h5uYBTvSsYvb+8j5mbJxfYg
K2A5QbqLEWwNkLxvWxH7sOQUDIWAr/iUK8v3qPsrjP9Fg49H/Gz0SiPn/6bi89g7hkkrfQv+K4Yq
cGbr/7MaNY4TfwDbd9AYja7LyQAAAABJRU5ErkJggg==">
  <p class="p-description">This is just a fun place to hang out virtually</p>
  <hr/>
  <p>Participants</p>
  <div class="e-participants">
    <p class="h-card" style="white-space: nowrap">
      <img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
      <span class="p-name">Alice Angel</span><br>
      Email: <span class="u-email">alice@example.com</span><br>
      PGP: <span class="u-key">...base64 key data...</span><br/>
      Birthday: <span class="dt-bday">--04-30</span><br>
      <span class="x-coi-status">working from home...</span>
    </p>
    <p class="h-card">
      <img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
      <span class="p-name">John Doe</span><br>
      Email: <span class="u-email">john.doe@example.com</span><br>
      PGP: <span class="u-key">...base64 key data...</span><br/>
    </p>
    <p class="h-card">
      <img class="u-photo" alt="" style="float: left" src="data:image/png;base64,
  iVBORw0KGgoAAAANSUhEUgAAACAAAAAgAQMAAABJtOi3AAAABlBMVEUAAAD///+l2Z/dAAAAOklE
  QVQI12P4DwQM6MQHeSDxgB9IHGAnSIDVgXWgmdLA/J/hHwPDf4Y/DAz1DD8YGOwhxAcGBnliCQCZ
  oU0qOMX4LwAAAABJRU5ErkJggg==">
      <span class="p-name">Caroll</span><br>
      Email: <span class="u-email">carrol@example.com</span><br>
      PGP: <span class="u-key">...base64 key data...</span><br/>
    </p>
  </div>
 </p>
 </div>
</body>
</html>

--A22090S.123213/example.com--
```

### Leaving a Group: Group Participant Left Message

Any user in a group can leave a group chat. 

When doing so, a COI client MUST send a group participant left message. The `Chat-Content` header field MUST be set to "groupleft". A COI client SHOULD include a short description about what happened, this MAY be a user-generated reason. A receiving COI client MUST update the internal represenation of the group and remove the leaving sender of a a `groupleft` message. The `Reply-To` header MAY be set to remaining group participants, so that users on non-COI-compatible clients will answer by default to only the remaining group participants.

A COI client SHOULD also send a `groupleft` message to a recipient that has been removed from a group by another user.

Note that a COI client user MAY receive further group messages after being removed from a group, for example when another group member was offline in between and had messages for that group waiting in a queue. A user might also be added at a later point again by another user. 

*Example for a group participant left message:*
```
From: Bob Barr <bob@example.com> 
To: alice@example.com; carrol@example.com; dean@example.com;
Subject: My Crew
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$group.1234abcd4321.Asd23212.A2328@example.com>
In-Reply-To: <coi$group.1234abcd4321.123213.213223@example.com>
References: <coi$group.1234abcd4321.238788.AB29702@example.com>
 <coi$group.1234abcd4321.123213.213223@example.com>
Content-Type: text/plain; charset=UTF-8
Chat-Version: 1.0
Chat-Content: groupleft
Reply-To: alice@example.com; carrol@example.com; dean@example.com;
MIME-Version: 1.0

hi guys, I think you can now roll this project on your own! Bye!
```


#### Converting a 1:1 to a Group Conversation
When a COI client wants to convert a 1:1 to a group conversation, it MUST create a corresponding group with at least accompanying new `References` and `Message-ID` header fields in the first message. COI clients SHOULD also sent fitting `groupinfo` message parts.

However, a non-COI-compatible client is not aware about the semantics and would basically keep the `References` and use a non-group-identifying `Message-ID` header field.

In that case, a COI client SHOULD generate a new group:
* The group name SHOULD be based on `Subject` field cleared any reply-notations like "RE:".
* The group participants are extracted from the `To` and `CC` fields.
* The group-ID should be generated.

When replying to such a message, a COI client SHOULD create a new thread by using no `References` header field, a new `Message-ID` header field and the group-name in the `Subject` field.

### Group Message Encryption
If the group conversation should be encrypted, the originating client MUST include the `key` field for each group participants' `h-card` element. If the public keys are known for all group participants, a COI client SHOULD encrypt messages for the group by default.

### Request Group Information
When a COI client lost the group information and cannot restore it neither from the *COI/CONTACTS* IMAP folder nor from the message history, then it MAY create the group definition based on a newly received message:
* The group-ID is detected from the `Message-ID` header field.
* The group-name is taken from the `Subject` header field.
* The participants are determinedfrom the `To` and `CC` header fields.

In that case, it is RECOMMENDED to request the group information. This is done by setting the `Chat-Content` header field to `grouprequested` with the next user-generated message. 

A receiving COI client in that group  SHOULD send the information with the next user-generated message.

*Example for requesting the group information:*
```
From: Alice <alice@example.com> 
To: bob@example.com; carrol@example.com; dean@example.com; 
Subject: My Crew
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$group.5422989123KJKK.S2571BC.2878&8@sample.com> 
Content-Type: text/plain; charset=UTF-8; format="flowed"
References: <coi$group.5422989123KJKK.434571BC.89A707D2@sample.com>
 <coi$group.5422989123KJKK.LK23299KJK.KHG8DHJ320LM@sample.com>
In-Reply-To: <coi$group.5422989123KJKK.LK23299KJK.KHG8DHJ320LM@sample.com>
Chat-Version: 1.0
Chat-Content: grouprequested
MIME-Version: 1.0
Disposition-Notification-To: Alice <alice@example.com> 

hey folks, I might be late, but I look forward to meet you today!
```


### Group Messaging and Mailing Lists
A mailing list discussion is from a clients view like a 1:1 chat with one person. Sometimes, users answer to both to the sender of a message and to the mailing list. It is RECOMMENDED that a COI client detects such duplicate messages and only shows the message list message, not additionally a 1:1 chat with the sender. This can be detected by looking at the first message-ID given in the `References` or `Reply-To` header fields. 

### Mentions in Group Messages
It is RECOMMENDED that COI clients allow to mention users and that mentions for the current users are highlighted.

Mentions follow the scheme `@"?NAME"?:EMAILADDRESS`, e.g. `@Gabe:gabe+private@example.com` or `@"Gabe Lastname":gabe+private@example.com`.

It is RECOMMENDED that COI clients only show the name-part of the mention text, not the full mention with the email address. Note that mentions can be done in both the plain text as well as the HTML part of a message.

In HTML the representation MUST be done with a link using the `mention` CSS class:
```
<a class="mention" href="mailto:gabe+private@example.com">@Gabe</a>
```
It is RECOMMENDED to show profile details of a mentioned participant, when a user interacts with the mention.

### Changing Group Definitions and Processing Changes
Any group member can update a group, i.e. change the name, description,  avatar or participants. So like email, a COI group chat has no hierarchy.

Hierarchical group communication will be added at a later point to COI with a *channels* concept, but this will require a COI-compatible IMAP server.

In case of group definition changes a COI client SHOULD update the corresponding group storage message, by `APPEND`-ing the new message,  then mark the original message with the `\Deleted` flag, and then `EXPUNGE`-ing deleted messages.

#### Processing Group Name Changes
As the group name is specified in the `Subject` field, it is easy for traditional email users to change the name. When the subject has changed, a COI client SHOULD display the new group name. It is RECOMMENDED to ignore typical reply notations such as a preceding "RE:", "AW:" (German), "SV:" (Svedish/Norwegian), etc. Compare https://en.wikipedia.org/wiki/List_of_email_subject_abbreviations for a list of typical reply abbreviations.

#### Processing Group Participants Changes
Group participants can be changed in different ways:
* By changing the recipients in the `To` or `CC` header fields of a group message. If a COI-compatile client  does this, a corresponding `Chat-Content: groupinfo` message part MUST be added, when one or more participants are added.
* By leaving the group with a `Chat-Content: groupleft` message.

When a participant is being added, clients SHOULD check if that participant has previously sent a `groupleft` message. In that case, the client SHOULD check if the `Reply-To` header of the adding message refers to a message older than the `groupleft` message. If that is the case, the re-addition of the user SHOULD be ignored.

#### Processing Group Description and Avatar Changes
Clients can also change the group description and avatar with a corresponding `groupinfo` message part at any time. Ideally, each change is accompanied by a user-generated message. A client SHOULD only use the latest `groupinfo` that has been received.

#### Processing Group Participant Changes
To protect a group participant from changes made by other participants, clients MUST use received participant information only, when they have not received the information from the participants directly. 

#### Processing Group Changes
*TODO: describe extracting participant information, generating respective contact storage messages or keeping the information within the group definition, re-generating the group information and then encrypting the message*

## Location Messages
Sometimes it is useful to share your location with other users. Clients SHOULD support sending and displaying location messages.

For transmitting the information, the [h-geo microformat](http://microformats.org/wiki/h-geo) is used which can be identified with a `Chat-Content: location` header field. It is RECOMMENDED to inlude an image that displays the area and the position of the user.

When receiving such a message, a client SHOULD provide an option to process this information further, for example by opening a corresponding mapping application.

*Example for a location message:*
```
From: Alice <alice@example.com> 
To: john.doe@example.com
Subject: Chat: 
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$232389K88.K2329KSOI@example.com> 
Content-Type: multipart/alternative; 
 boundary="BKKHJ222.123213/example.com"
Chat-Version: 1.0
MIME-Version: 1.0

--BKKHJ222.123213/example.com
Content-Type: text/plain; charset=UTF-8; format="flowed"

Let's meet here.
-
Position: 53.07516, 8.80777

--BKKHJ222.123213/example.com
Content-Type: text/html; charset=UTF-8
Chat-Content: location

<html>
<body>
 <p>Let's meet here.</p>
 <hr/>
 <p>Position:</p>
 <p class="h-geo">
  <span class="p-latitude">53.07516</span>,
  <span class="p-longitude">8.80777</span>
</p>
</body>
</html>

--BKKHJ222.123213/example.com--
```


## Pinned Messages
To allow highlighting important messages, any chat message may be pinned. To achieve this, the `Content-Disposition` header field is set to "pinned". Optionally, a "timeout" parameter can be specified that contains a datetime until the message should be pinned.

COI clients SHOULD keep a pinned message in visible view area until:

* The specified timeout occurred, or
* a new pinned message arrived in a conversation, or
* the user locally unpinned a message.

*Example for a pinned message:*
```
From: Me Myself <me@example.com> 
To: You <you@recipientdomain.com> 
Subject: Chat: Hello COI...
Date: Mon, 4 Dec 2019 15:51:37 +0100 
Message-ID: <coi$S2571BC.2878&8@sample.com> 
Content-Type: text/plain; charset=UTF-8 
Content-Disposition: pinned;
 timeout=Tue, 5 Dec 2019 0:00:00 +0100 
References: <coi$434571BC.89A707D2@sample.com> 
Chat-Version: 1.0
MIME-Version: 1.0
Disposition-Notification-To: Me Myself <me@example.com> 

Hello COI world, this message is pinned!
```

## Poll Messages
Polls are an integral part of social communication, e.g. for finding a suitable date or selecting a preferred option. To provide support for this interaction, COI clients SHOULD support displaying and voting poll messages.

*TODO describe poll messages in more detail*/



## Extension Messages
COI clients MAY introduce client specific messages. Those messages SHOULD always include a text/plain,  text/html or binary inline part explaining the contents for users of non COI compliant clients. COI clients MAY use the "Chat-Content" header field for their data message parts. To ensure uniqueness of "Chat-Content" header values, clients MUST start the value with a client-specific name. A reverse domain for the value start is RECOMMENDED, for example:
```
Chat-Content: com.awesomeclient.mycustomevent
```

# Message Encryption
A COI-compliant client SHOULD support the [Autocrypt.org](https://autocrypt.org) standard to ease end to end encryption scenarios.

Further encryption mechanisms will be defined in the future.


# Separate COI and Mail Messages
There are different scenarios for handling mail messages:
* Some clients may treat all mail messages the same and provide a chat-centric experience for each one.
* Other clients may want to differentiate between a traditional mail experience and a chat experience depending on the message or contact type.

Depending on the client's and user's needs, clients MAY offer one of the following differentiation options:

1. No differentiation, every new message ends up by default in the *Inbox*.
2. Strict separation, chat messages will be moved to the *COI/CHATS* folder.
3. Separation after having read. Chat messages will be moved to the *COI/CHATS* folder after having marked read.

### Configuring Separation on COI Servers
On a COI-compliant server, the separation can be automated by the server. Note that depending on configuration values returned by the `ENABLE COI` IMAP command, a different folder prefix than *COI* may be used.

To configure the filtering, `APPEND` a message in the *COI/CONFIGURATION* folder that specifies filtering with the `COI-MESSAGE-FILTER` header field. 

Possible value are:
* `active`: chat and email messages are sorted 
* `none`: chat messages stay in Inbox
* `read`: only read chat messages are sorted

*Example for an active filtering:*
```
From: user@domain
Date: Mon, 11 Feb 2019 15:31:78 +0100 
COI-Message-Filter: active
```
In case several messages exist with the `COI-MESSAGE-FILTER` header, only the newest message will be considered.

### Manual Message Seperation

Depending on the above scenarios, COI clients MAY move chat messages to the *COI/CHAT*S folder. In that case, replies from non-COI compliant clients to COI messages SHOULD be moved to the *COI/CHATS* folder as well.

Replies of non-COI compliant clients can be identified as chat message by having:

* Having a reference that starts with a message ID starting with `<coi$`, or
* Having an `In-Reply-To` header field with a message-ID that starts with `<coi$`.

For moving the message to COI/CHATS a COI client has the following options:

* Client listens for incoming mails on *INBOX* either using IMAP IDLE or IMAP NOTIFY, and then moves matching chat mails manually to *COI/CHATS*.
* Pending the user's approval, a service could monitor incoming mails and do that.

After moving a message that originates from the user itself, identified either with the primary email address or a valid alias address, this message should be marked as read at the same time.

*TODO describe how to do that.*

# Folders
When a COI client connects to a non-COI IMAP server, it might need to create several folders initially:

* *COI/CHATS* for storing all chat messages. This folder SHOULD be subscribed the user.
* *COI/CONTACTS* for storing contacts.
* *BLOCKED* for storing messages originating from blocked contacts.

*TODO describe IMAP details*

# Security Considerations
The security consideration of the referenced standards also apply to this COI client specification definition.

# IANA Considerations
The "pinned" value of the "Content-Disposition" header field should be registered with IANA. 