---
title: Using Datagrams with HTTP
abbrev: HTTP Datagrams
docname: draft-ietf-masque-h3-datagram-latest
category: std
wg: MASQUE

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "D. Schinazi"
    name: "David Schinazi"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: dschinazi.ietf@gmail.com

 -
    ins: "L. Pardue"
    name: "Lucas Pardue"
    organization: "Cloudflare"
    email: lucaspardue.24.7@gmail.com

informative:
  PRIORITY: I-D.ietf-httpbis-priority

--- abstract

The QUIC DATAGRAM extension provides application protocols running over QUIC
with a mechanism to send unreliable data while leveraging the security and
congestion-control properties of QUIC. However, QUIC DATAGRAM frames do not
provide a means to demultiplex application contexts. This document describes how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3. It associates datagrams with client-initiated bidirectional streams
and defines an optional additional demultiplexing layer. Additionally, this
document defines how to convey datagrams over prior versions of HTTP.


--- middle

# Introduction {#intro}

The QUIC DATAGRAM extension {{!DGRAM=I-D.ietf-quic-datagram}} provides
application protocols running over QUIC {{!QUIC=RFC9000}} with a mechanism to
send unreliable data while leveraging the security and congestion-control
properties of QUIC. However, QUIC DATAGRAM frames do not provide a means to
demultiplex application contexts. This document describes how to use QUIC
DATAGRAM frames when the application protocol running over QUIC is HTTP/3
{{!H3=I-D.ietf-quic-http}}. It associates datagrams with client-initiated
bidirectional streams and defines an optional additional demultiplexing layer.
Additionally, this document defines how to convey datagrams over prior versions
of HTTP.

This document is structured as follows:

* {{multiplexing}} presents core concepts for multiplexing across HTTP versions.
  * {{datagram-contexts}} defines datagram contexts, an optional end-to-end
    multiplexing concept scoped to each HTTP request.
  * Contexts are identified using a variable-length integer. Requirements for
    allocating identifier values are detailed in {{context-id-alloc}}.
* {{format}} defines how QUIC DATAGRAM frames are used with HTTP/3. {{setting}}
  defines an HTTP/3 setting that endpoints can use to advertise support of the
  frame.
* {{capsule}} introduces the Capsule Protocol and the "data stream" concept.
  Data streams are initiated using special-purpose HTTP requests, after which
  Capsules, an end-to-end message, can be sent.
  * {{datagram-capsule}} defines Datagram Capsule types, along with guidance
    for specifying new capsule types.


## Conventions and Definitions {#defs}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Multiplexing

When running over HTTP/3, multiple exchanges of datagrams need the ability to
coexist on a given QUIC connection. To allow this, HTTP datagrams contain two
layers of multiplexing. First, the QUIC DATAGRAM frame payload starts with an
encoded stream identifier that associates the datagram with a given QUIC
stream. Second, datagrams optionally carry a context identifier (see
{{datagram-contexts}}) that allows multiplexing multiple datagram contexts
related to a given HTTP request. Conceptually, the first layer of multiplexing
is per-hop, while the second is end-to-end.

When running over HTTP/2, the first level of demultiplexing is provided by the
HTTP/2 framing layer. When running over HTTP/1, requests are strictly
serialized in the connection, therefore the first layer of demultiplexing is
not needed.


## Datagram Contexts {#datagram-contexts}

Within the scope of a given HTTP request, contexts provide an additional
demultiplexing layer. Contexts determine the encoding of datagrams, and can be
used to implicitly convey metadata. For example, contexts can be used for
compression to elide some parts of the datagram: the context identifier then
maps to a compression context that the receiver can use to reconstruct the
elided data.

Any HTTP Methods or protocols enabled by HTTP Upgrade Tokens that use HTTP
Datagrams MUST define the format of datagrams for the default context, which
has a context ID of 0.

Non-default contexts (contexts with a non-zero ID) are OPTIONAL to implement
for both endpoints. Intermediaries do not require any context-specific software
to enable such contexts. Methods or protocols that use HTTP Datagrams can
independently define whether or not non-default contexts are supported, and
how to negotiate support for contexts if needed. Such methods or protocols
can define new HTTP Capsule types ({{capsule-protocol}}) to register
a context ID and indicate the meaning or format of datagrams that have that
context ID. For guidance on registering contexts with capsules, see
{{register-capsules}}.

While stream IDs are a per-hop concept, context IDs are an end-to-end concept.
In other words, if a datagram travels through one or more intermediaries on its
way from client to server, the stream ID will most likely change from hop to
hop, but the context ID will remain the same. Context IDs are opaque to
intermediaries.

When contexts are used, they are identified within the scope of a given request
by a numeric value, referred to as the context ID. A context ID is a 62-bit
integer (0 to 2<sup>62</sup>-1).


## Context ID Allocation {#context-id-alloc}

Implementations of HTTP Datagrams that support datagram contexts MUST provide
a context ID allocation service. That service will allow applications
co-located with HTTP to request a unique context ID that they can subsequently
use for their own purposes. The HTTP implementation will then parse the context
ID of incoming HTTP Datagrams and use it to deliver the frame to the
appropriate application context.

Even-numbered context IDs are client-initiated, while odd-numbered context IDs
are server-initiated. This means that an HTTP client implementation of the
context ID allocation service MUST only provide even-numbered IDs, while a
server implementation MUST only provide odd-numbered IDs. Note that, once
allocated, any context ID can be used by both client and server - only
allocation carries separate namespaces to avoid requiring synchronization.
Additionally, note that the context ID namespace is tied to a given HTTP
request: it is possible for the same numeral context ID to be used
simultaneously in distinct requests.


# HTTP/3 DATAGRAM Format {#format}

When used with HTTP/3, the Datagram Data field of QUIC DATAGRAM frames uses the
following format (using the notation from the "Notational Conventions" section
of {{QUIC}}):

~~~
HTTP/3 Datagram {
  Encoded Stream ID (i),
  [Context ID (i)],
  HTTP Datagram Payload (..),
}
~~~
{: #h3-datagram-format title="HTTP/3 DATAGRAM Format"}

Encoded Stream ID:

: A variable-length integer that contains an encoded value indicating the
client-initiated bidirectional stream that this datagram is associated with.
The Encoded Stream ID for a datagram without a context ID is the associated
stream ID divided by two. The Encoded Stream ID for a datagram with a context
ID is the associated stream ID divided by two, plus one. (Client-initiated
bidirectional streams have stream IDs that are divisible by four, so the
stream ID divided by two always has the least-significant bit set to zero).
The largest legal QUIC stream ID value is 2<sup>62</sup>-1, so the largest legal
value of Encoded Stream ID is (2<sup>62</sup>-1 / 2) + 1. Receipt of a frame that
includes a larger value MUST be treated as a connection error of type
FRAME_ENCODING_ERROR. A datagram without a context ID is implicitly using
a context ID of 0.

Context ID:

: A variable-length integer indicating the context ID of the datagram (see
{{datagram-contexts}}). Whether or not this field is present depends on whether
the least-significant bit of the Encoded Stream ID is set, as described above. If the
receiver of a datagram does not support Context IDs, it will drop any datagram
that contains a context ID. If the Context ID field is not present, this datagram
is associated with the default context (which has its Context ID set to 0).

HTTP Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Intermediaries parse the Encoded Stream ID field in order to associate the QUIC
DATAGRAM frame with a stream. This is done by first checking if the
least-significant bit of the Encoded Stream ID is set (which indicates if the
datagram includes a context ID), clearing that bit, and multiplying the result
by two to generate the client-initiated bidirectional stream ID. If an
intermediary receives a QUIC DATAGRAM frame whose payload is too short to
allow parsing the Encoded Stream ID field, the intermediary MUST treat it as
an HTTP/3 connection error of type H3_GENERAL_PROTOCOL_ERROR.

Endpoints parse both the Encoded Stream ID field and the Context ID field in
order to associate the QUIC DATAGRAM frame with a stream and context within
that stream. If an endpoint receives a QUIC DATAGRAM frame whose payload is too
short to allow parsing the Encoded Stream ID field, the endpoint MUST treat it
as an HTTP/3 connection error of type H3_GENERAL_PROTOCOL_ERROR. If an endpoint
receives a QUIC DATAGRAM frame whose payload is long enough to allow parsing
the Encoded Stream ID field but too short to allow parsing the Context ID
field, the endpoint MUST abruptly terminate the corresponding stream with a
stream error of type H3_GENERAL_PROTOCOL_ERROR.

Endpoints MUST NOT send HTTP/3 datagrams unless the corresponding stream's send
side is open. On a given endpoint, once the receive side of a stream is closed,
incoming datagrams for this stream are no longer expected so the endpoint can
release related state. Endpoints MAY keep state for a short time to account for
reordering. Once the state is released, the endpoint MUST silently drop
received associated datagrams.

If an HTTP/3 datagram is received and its Encoded Stream ID maps to a stream
that has not yet been created, the receiver SHALL either drop that datagram
silently or buffer it temporarily while awaiting the creation of the
corresponding stream.


# Capsules {#capsule}

This specification introduces the Capsule Protocol. The Capsule Protocol is a
sequence of type-length-value tuples that allows endpoints to reliably
communicate request-related information end-to-end, even in the presence of
HTTP intermediaries.


## Capsule Protocol {#capsule-protocol}

This specification defines the "data stream" of an HTTP request as the
bidirectional stream of bytes that follow the headers in both directions. In
HTTP/1.x, the data stream consists of all bytes on the connection that follow
the blank line that concludes either the request header section, or the 2xx
(Successful) response header section. In HTTP/2 and HTTP/3, the data stream of
a given HTTP request consists of all bytes sent in DATA frames with the
corresponding stream ID. The concept of a data stream is particularly relevant
for methods such as CONNECT where there is no HTTP message content after the
headers.

Definitions of new HTTP Methods or of new HTTP Upgrade Tokens can state that
their data stream uses the Capsule Protocol. If they do so, that means that the
contents of their data stream uses the following format (using the notation
from the "Notational Conventions" section of {{QUIC}}):

~~~
Capsule Protocol {
  Capsule (..) ...,
}
~~~
{: #capsule-stream-format title="Capsule Protocol Stream Format"}

~~~
Capsule {
  Capsule Type (i),
  Capsule Length (i),
  Capsule Value (..),
}
~~~
{: #capsule-format title="Capsule Format"}

Capsule Type:

: A variable-length integer indicating the Type of the capsule. Endpoints that
receive a capsule with an unknown Capsule Type MUST silently skip over that
capsule.

Capsule Length:

: The length of the Capsule Value field following this field, encoded as a
variable-length integer. Note that this field can have a value of zero.

Capsule Value:

: The payload of this capsule. Its semantics are determined by the value of the
Capsule Type field.


## Requirements

If the definition of an HTTP Method or HTTP Upgrade Token states that it uses
the capsule protocol, its implementations MUST follow the following
requirements:

* A server MUST NOT send any Transfer-Encoding or Content-Length header fields
  in a 2xx (Successful) response. If a client receives a Content-Length or
  Transfer-Encoding header fields in a successful response, it MUST treat that
  response as malformed.

* A request message does not have content.

* A successful response message does not have content.

* Responses are not cacheable.

## Intermediary Processing

Intermediaries MUST operate in one of the two following modes:

Pass-through mode:

: In this mode, the intermediary forwards the data stream between two
associated streams without any modification of the data stream.

Participant mode:

: In this mode, the intermediary terminates the data stream and parses all
Capsule Type and Capsule Length fields it receives.

Each Capsule Type determines whether it is opaque or transparent to
intermediaries in participant mode: opaque capsules are forwarded unmodified
while transparent ones can be parsed, added, or removed by intermediaries.
Intermediaries MAY modify the contents of the Capsule Data field of transparent
capsule types.

Unless otherwise specified, all Capsule Types are defined as opaque to
intermediaries. Intermediaries MUST forward all received opaque CAPSULE frames
in their unmodified entirety. Intermediaries MUST NOT send any opaque CAPSULE
frames other than the ones it is forwarding. All Capsule Types defined in this
document are opaque, with the exception of the datagram capsules, see
{{datagram-capsule}}. Definitions of new Capsule Types MAY specify that the
newly introduced type is transparent. Intermediaries MUST treat unknown Capsule
Types as opaque.

Intermediaries respect the order of opaque CAPSULE frames: if an intermediary
receives two opaque CAPSULE frames in a given order, it MUST forward them in
the same order.

Endpoints which receive a Capsule with an unknown Capsule Type MUST silently
drop that Capsule.


## The Datagram Capsules {#datagram-capsule}

This document defines the DATAGRAM and DATAGRAM_WITH_CONTEXT capsules types,
known collectively as the datagram capsules (see {{iana-types}} for the value
of the capsule types). These capsules allow an endpoint to send a datagram
frame over an HTTP stream. This is particularly useful when using a version of
HTTP that does not support QUIC DATAGRAM frames.

~~~
Datagram Capsule {
  Type (i) = DATAGRAM or DATAGRAM_WITH_CONTEXT,
  Length (i),
  [Context ID (i)],
  HTTP Datagram Payload (..),
}
~~~
{: #datagram-capsule-format title="DATAGRAM Capsule Format"}

Context ID:

: A variable-length integer indicating the context ID of the datagram (see
{{datagram-contexts}}). This field is present in DATAGRAM_WITH_CONTEXT capsules
but not in DATAGRAM capsules. If a DATAGRAM capsule is used on a stream where
datagram contexts are in use, it is associated with context ID 0.
DATAGRAM_WITH_CONTEXT capsules MUST NOT carry context ID 0 as that context ID
is conveyed using the DATAGRAM capsule.

HTTP Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Datagrams sent using the datagram capsule have the exact same semantics as
datagrams sent in QUIC DATAGRAM frames. In particular, the restrictions on when
it is allowed to send an HTTP Datagram and how to process them from {{format}}
also apply to HTTP Datagrams sent and received using the datagram capsules.

The datagram capsules are transparent to intermediaries, meaning that
intermediaries MAY parse them and send datagram capsules that they did not
receive. This allows an intermediary to reencode HTTP Datagrams as it forwards
them: in other words, an intermediary MAY send a datagram capsule to forward an
HTTP Datagram which was received in a QUIC DATAGRAM frame, and vice versa.

Note that while datagram capsules are sent on a stream, intermediaries can
reencode HTTP Datagrams into QUIC DATAGRAM frames over the next hop, and those
could be dropped. Because of this, applications have to always consider HTTP
Datagrams to be unreliable, even if they were initially sent in a capsule.

If an intermediary receives an HTTP Datagram in a QUIC DATAGRAM frame and is
forwarding it on a connection that supports QUIC DATAGRAM frames, the
intermediary SHOULD NOT convert that HTTP Datagram to a DATAGRAM capsule. If
the HTTP Datagram is too large to fit in a DATAGRAM frame (for example because
the path MTU of that QUIC connection is too low or if the maximum UDP payload
size advertised on that connection is too low), the intermediary SHOULD drop
the HTTP Datagram instead of converting it to a DATAGRAM capsule. This
preserves the end-to-end unreliability characteristic that methods such as
Datagram Packetization Layer Path MTU Discovery (DPLPMTUD) depend on
{{?RFC8899}}. An intermediary that converts QUIC DATAGRAM frames to datagram
capsules allows HTTP Datagrams to be arbitrarily large without suffering any
loss; this can misrepresent the true path properties, defeating methods such a
DPLPMTUD.


## Registering Datagram Contexts with Capsules {#register-capsules}

Methods or protocols that support multiple datagram payload formats or
separate types of datagrams can differentiate them using datagram contexts
({{datagram-contexts}}).

Such protocols can define a new Capsule type that is used to register
a context ID with the peer endpoint.

For example, if a new method needed to define a non-default context for
the "Example" datagram format, it could define a new capsule type
(REGISTER_EXAMPLE_FORMAT) that indcludes a context ID value. Endpoints
that understand this new capsule type would be able to consequently
handle and parse datagrams on the context ID, while all other endpoints
would ignore the datagrams.

~~~
REGISTER_EXAMPLE_FORMAT Capsule {
  Type (i) = REGISTER_EXAMPLE_FORMAT, // For example only
  Length (i),
  Context ID (i),
}
~~~
{: #example-capsule-format title="REGISTER_EXAMPLE_FORMAT Capsule Format"}


# The H3_DATAGRAM HTTP/3 SETTINGS Parameter {#setting}

Implementations of HTTP/3 that support HTTP Datagrams can indicate that to
their peer by sending the H3_DATAGRAM SETTINGS parameter with a value of 1.
The value of the H3_DATAGRAM SETTINGS parameter MUST be either 0 or 1. A value
of 0 indicates that HTTP Datagrams are not supported. An endpoint that receives
the H3_DATAGRAM SETTINGS parameter with a value that is neither 0 or 1 MUST
terminate the connection with error H3_SETTINGS_ERROR.

Endpoints MUST NOT send QUIC DATAGRAM frames until they have both sent and
received the H3_DATAGRAM SETTINGS parameter with a value of 1.

When clients use 0-RTT, they MAY store the value of the server's H3_DATAGRAM
SETTINGS parameter. Doing so allows the client to send QUIC DATAGRAM frames in
0-RTT packets. When servers decide to accept 0-RTT data, they MUST send a
H3_DATAGRAM SETTINGS parameter greater than or equal to the value they sent to
the client in the connection where they sent them the NewSessionTicket message.
If a client stores the value of the H3_DATAGRAM SETTINGS parameter with their
0-RTT state, they MUST validate that the new value of the H3_DATAGRAM SETTINGS
parameter sent by the server in the handshake is greater than or equal to the
stored value; if not, the client MUST terminate the connection with error
H3_SETTINGS_ERROR. In all cases, the maximum permitted value of the H3_DATAGRAM
SETTINGS parameter is 1.


## Note About Draft Versions

\[\[RFC editor: please remove this section before publication.]]

Some revisions of this draft specification use a different value (the
Identifier field of a Setting in the HTTP/3 SETTINGS frame) for the H3_DATAGRAM
Settings Parameter. This allows new draft revisions to make incompatible
changes. Multiple draft versions MAY be supported by either endpoint in a
connection. Such endpoints MUST send multiple values for H3_DATAGRAM. Once an
endpoint has sent and received SETTINGS, it MUST compute the intersection of
the values it has sent and received, and then it MUST select and use the most
recent draft version from the intersection set. This ensures that both
endpoints negotiate the same draft version.


# Prioritization

Data streams (see {{capsule-protocol}}) can be prioritized using any means
suited to stream or request prioritization. For example, see {{Section 11 of
PRIORITY}}.

Prioritization of HTTP/3 datagrams is not defined in this document. Future
extensions MAY define how to prioritize datagrams, and MAY define signaling to
allow endpoints to communicate their prioritization preferences.


# Security Considerations {#security}

Since this feature requires sending an HTTP/3 Settings parameter, it "sticks
out". In other words, probing clients can learn whether a server supports this
feature. Implementations that support this feature SHOULD always send this
Settings parameter to avoid leaking the fact that there are applications using
HTTP/3 datagrams enabled on this endpoint.


# IANA Considerations {#iana}

## HTTP/3 SETTINGS Parameter {#iana-setting}

This document will request IANA to register the following entry in the
"HTTP/3 Settings" registry:

| Setting Name |   Value  | Specification | Default |
|:-------------|:---------|:--------------|:--------|
| H3_DATAGRAM  | 0xffd277 | This Document |    0    |
{: #iana-setting-table title="New HTTP/3 Settings"}


## HTTP Header Field Name {#iana-hdr}

This document will request IANA to register the following entry in the
"HTTP Field Name" registry:

Field Name:

: Sec-Use-Datagram-Contexts

Template:

: None

Status:

: provisional (permanent if this document is approved)

Reference:

: This document

Comments:

: None


## Capsule Types {#iana-types}

This document establishes a registry for HTTP capsule type codes. The "HTTP
Capsule Types" registry governs a 62-bit space. Registrations in this registry
MUST include the following fields:

Type:

: A name or label for the capsule type.

Value:

: The value of the Capsule Type field (see {{capsule-protocol}}) is a 62-bit
integer.

Reference:

: An optional reference to a specification for the type. This field MAY be
empty.

Registrations follow the "First Come First Served" policy (see Section 4.4 of
{{!IANA-POLICY=RFC8126}}) where two registrations MUST NOT have the same Type.

This registry initially contains the following entries:

| Capsule Type                 |   Value   | Specification |
|:-----------------------------|:----------|:--------------|
| DATAGRAM                     | 0xff37a1  | This Document |
| DATAGRAM_WITH_CONTEXT        | 0xff37a2  | This Document |
{: #iana-types-table title="Initial Capsule Types Registry Entries"}

Capsule types with a value of the form 41 * N + 23 for integer values of N are
reserved to exercise the requirement that unknown capsule types be ignored.
These capsules have no semantics and can carry arbitrary values. These values
MUST NOT be assigned by IANA and MUST NOT appear in the listing of assigned
values.



--- back

# Examples

## CONNECT-UDP

In this example, the client does not support any CONNECT-UDP nor HTTP Datagram
extensions, and therefore has no use for datagram contexts on this stream.
Each datagram payload is expected to contain a UDP payload.

~~~
Client                                             Server

STREAM(44): HEADERS             -------->
  :method = CONNECT
  :protocol = connect-udp
  :scheme = https
  :path = /target.example.org/443/
  :authority = proxy.example.org:443

DATAGRAM                        -------->
  Encoded Stream ID = 22
  Payload = Encapsulated UDP Payload

           <--------  STREAM(44): HEADERS
                        :status = 200

/* Wait for target server to respond to UDP packet. */

           <--------  DATAGRAM
                        Encoded Stream ID = 22
                        Payload = Encapsulated UDP Payload
~~~


## CONNECT-UDP with Timestamp Extension

In these examples, the client supports a CONNECT-UDP Timestamp Extension, which
uses a different Datagram Format Type that carries a timestamp followed by the
encapsulated UDP payload. Datagrams on the default context (0) are expected to
contain UDP payloads, while contexts established with the
REGISTER_UDP_WITH_TIMESTAMP_CONTEXT capsule include a timestamp along with the
UDP payload. A new header, Sec-UDP-Timestamps, indicates support for the
extension.


### Extension Supported

In this instance, the server supports non-default datagram contexts for the
timestamp extension. The client registers a context for UDP payloads
with timestamps once it learns that the extension is supported.

~~~
Client                                             Server

STREAM(44): HEADERS            -------->
  :method = CONNECT
  :protocol = connect-udp
  :scheme = https
  :path = /target.example.org/443/
  :authority = proxy.example.org:443
  Sec-UDP-Timestamps = ?1

DATAGRAM                        -------->
  Encoded Stream ID = 22
  Payload = Encapsulated UDP Payload

           <--------  STREAM(44): HEADERS
                        :status = 200
                        Sec-UDP-Timestamps = ?1

/* UDP timestamps are supported on this stream */

STREAM(44): DATA               -------->
  Capsule Type = REGISTER_UDP_WITH_TIMESTAMP_CONTEXT
  Context ID = 2

DATAGRAM                        -------->
  Encoded Stream ID = 23
  Context ID = 2
  Payload = Encapsulated UDP Payload With Timestamp

           <--------  DATAGRAM
                        Encoded Stream ID = 22
                        Payload = Encapsulated UDP Payload
~~~


### Extension Unsupported

In this instance, the server does not support non-default datagram contexts
for the timestamp extension.


~~~
Client                                             Server

STREAM(44): HEADERS            -------->
  :method = CONNECT
  :protocol = connect-udp
  :scheme = https
  :path = /target.example.org/443/
  :authority = proxy.example.org:443
  Sec-UDP-Timestamps = ?1

DATAGRAM                        -------->
  Encoded Stream ID = 22
  Payload = Encapsulated UDP Payload

           <--------  STREAM(44): HEADERS
                        :status = 200

/* UDP timestamps are not supported on this stream */

           <--------  DATAGRAM
                        Encoded Stream ID = 22
                        Payload = Encapsulated UDP Payload
~~~


## CONNECT-IP with IP compression

~~~
Client                                             Server

STREAM(44): HEADERS            -------->
  :method = CONNECT
  :protocol = connect-ip
  :scheme = https
  :path = /
  :authority = proxy.example.org:443
  Sec-Use-IP-Compression = ?1

           <--------  STREAM(44): HEADERS
                        :status = 200
                        Sec-Use-IP-Compression = ?1

/* Exchange CONNECT-IP configuration information. */

DATAGRAM                       -------->
  Encoded Stream ID = 22
  Payload = Encapsulated IP Packet

/* Endpoint happily exchange encapsulated IP packets */
/* using Encoded Stream ID 22 and Context ID 0.      */

DATAGRAM                       -------->
  Encoded Stream ID = 22
  Payload = Encapsulated IP Packet

/* After performing some analysis on traffic patterns, */
/* the client decides it wants to compress a 2-tuple.  */


STREAM(44): DATA                -------->
  Capsule Type = REGISTER_IP_COMPRESSION_CONTEXT
  Context ID = 2
  Compression Tuple = "192.0.2.6,192.0.2.7"

DATAGRAM                       -------->
  Encoded Stream ID = 23
  Context ID = 2
  Payload = Compressed IP Packet
~~~


## WebTransport

~~~
Client                                             Server

STREAM(44): HEADERS            -------->
  :method = CONNECT
  :scheme = https
  :method = webtransport
  :path = /hello
  :authority = webtransport.example.org:443
  Origin = https://www.example.org:443

           <--------  STREAM(44): HEADERS
                        :status = 200

/* Both endpoints can now send WebTransport datagrams. */
~~~


# Acknowledgments {#acks}
{:numbered="false"}

The DATAGRAM context identifier was previously part of the DATAGRAM frame
definition itself, the authors would like to acknowledge the authors of that
document and the members of the IETF MASQUE working group for their
suggestions. Additionally, the authors would like to thank Martin Thomson for
suggesting the use of an HTTP/3 SETTINGS parameter. Furthermore, the authors
would like to thank Ben Schwartz for writing the first proposal that used two
layers of indirection.
