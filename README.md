# COI-Specs
The open specification for COI compatible clients.

* [COI Client Spec](coi-client-spec.md)
* [SMTP: Submission Token Extension Draft](https://www.ietf.org/staging/draft-ietf-extra-smtp-submission-token-00.txt)

# What is COI
COI is an open chat communication standard built on top of IMAP and SMTP.  
COI is a combination of the COI Standard plus compatible Client Apps and Email Servers. 
COI works with all email servers, but IMAP servers can be enhanced with extra COI capabilities.  

# COI Principles
The principle of COI is very simple: COI uses IMAP/SMTP as the transport mechanism for a chat application.
The generic steps are as follows:  
	1.	Establish a connection with an SMTP server (Send) and an IMAP server (Receive)  
	2.	Establish the capabilities of the IMAP server. This determines how you communicate with it  
	3.	Send chat messages over SMTP, receive messages over IMAP  

# COI Now & Soon
Today, you can use COI to communicate with other compatible messengers over existing IMAP and SMTP servers.
Soon, additional functionalities will be made available by COI compatible IMAP servers. Example for such features are automated filtering between normal mail messages and chat messages, server-side blocking of contacts, support for push notifications, channels, WebRTC and much more.

# Required Knowledge
To build a COI application you need to know the following:  
	•	How to communicate with and IMAP and SMTP server - This is usually done by using a native email library.  
	•	A good understanding of the COI protocol in both its variants (COI over simple IMAP servers and COI over COI extended IMAP servers).  
	•	Know how to build your desired application.  
	•	A basic understanding of the structure of SMTP/IMAP messages - Specifically an working knowledge of RFC 5322, but reading the COI protocol should give you most of what you need.

# What you need
As a client developer, you have several options:  
	•	Communicate with the email server using an IMAP library for your preferred language & platform – or even communicate directly when you feel adventurous.  
	•	Base your work on Delta Chat Core, an MPL licensed library that abstracts away IMAP communication and that will become COI compliant.  
	•	Base your work on an existing COI compatible app, like the cross-platform OX Talk app.  
	
Visist https://coi-dev.org for more details.
