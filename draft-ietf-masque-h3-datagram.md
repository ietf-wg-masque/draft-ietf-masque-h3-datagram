---
title: Using Datagrams with HTTP
abbrev: HTTP Datagrams
docname: draft-ietf-masque-h3-datagram-latest
submissiontype: IETF
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


--- abstract

The QUIC DATAGRAM extension provides application protocols running over QUIC
with a mechanism to send unreliable data while leveraging the security and
congestion-control properties of QUIC. However, QUIC DATAGRAM frames do not
provide a means to demultiplex application contexts. This document describes how
to use QUIC DATAGRAM frames with HTTP/3 by association with HTTP requests.
Additionally, this document defines the Capsule Protocol that can convey
datagrams over prior versions of HTTP.


--- middle

# Introduction {#intro}

The QUIC DATAGRAM extension {{!DGRAM=I-D.ietf-quic-datagram}} provides
application protocols running over QUIC {{!QUIC=RFC9000}} with a mechanism to
send unreliable data while leveraging the security and congestion-control
properties of QUIC. However, QUIC DATAGRAM frames do not provide a means to
demultiplex application contexts. This document describes how to use QUIC
DATAGRAM frames with HTTP/3 {{!H3=I-D.ietf-quic-http}} by association with HTTP
requests. Additionally, this document defines the Capsule Protocol that can
convey datagrams over prior versions of HTTP.


This document is structured as follows:

* {{multiplexing}} presents core concepts for multiplexing across HTTP versions.
* {{format}} defines how QUIC DATAGRAM frames are used with HTTP/3.
  * {{setting}} defines an HTTP/3 setting that endpoints can use to advertise
  support of the frame.
* {{capsule}} introduces the Capsule Protocol and the "data stream" concept.
  Data streams are initiated using special-purpose HTTP requests, after which
  Capsules, an end-to-end message, can be sent.
  * {{datagram-capsule}} defines Datagram Capsule types, along with guidance
    for specifying new capsule types.


## Conventions and Definitions {#defs}

{::boilerplate bcp14-tagged}


# Multiplexing

All HTTP Datagrams are associated with an HTTP request.

When running over HTTP/3, multiple exchanges of datagrams need the ability to
coexist on a given QUIC connection. To allow this, the QUIC DATAGRAM frame
payload starts with an encoded stream identifier that associates the datagram
with a request stream.

When running over HTTP/2, demultiplexing is provided by the HTTP/2 framing
layer. When running over HTTP/1, requests are strictly serialized in the
connection, therefore demultiplexing is not needed.


# HTTP/3 Datagram Format {#format}

When used with HTTP/3, the Datagram Data field of QUIC DATAGRAM frames uses the
following format (using the notation from the "Notational Conventions" section
of {{QUIC}}):

~~~
HTTP/3 Datagram {
  Quarter Stream ID (i),
  HTTP Datagram Payload (..),
}
~~~
{: #h3-datagram-format title="HTTP/3 DATAGRAM Format"}

Quarter Stream ID:

: A variable-length integer that contains the value of the client-initiated
bidirectional stream that this datagram is associated with, divided by four (the
division by four stems from the fact that HTTP requests are sent on
client-initiated bidirectional streams, and those have stream IDs that are
divisible by four). The largest legal QUIC stream ID value is 2<sup>62</sup>-1,
so the largest legal value of Quarter Stream ID is 2<sup>60</sup>-1. Receipt of
a frame that includes a larger value MUST be treated as an HTTP/3 connection
error of type H3_DATAGRAM_ERROR.

HTTP Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Receipt of a QUIC DATAGRAM frame whose payload is too short to allow parsing the
Quarter Stream ID field MUST be treated as an HTTP/3 connection error of type
H3_DATAGRAM_ERROR.

Endpoints MUST NOT send HTTP/3 datagrams unless the corresponding stream's send
side is open. On a given endpoint, once the receive side of a stream is closed,
incoming datagrams for this stream are no longer expected so the endpoint can
release related state. Endpoints MAY keep state for a short time to account for
reordering. Once the state is released, the endpoint MUST silently drop
received associated datagrams.

If an HTTP/3 datagram is received and its Quarter Stream ID maps to a stream
that has not yet been created, the receiver SHALL either drop that datagram
silently or buffer it temporarily while awaiting the creation of the
corresponding stream.

If an HTTP/3 datagram is received and its Quarter Stream ID maps to a stream
that cannot be created due to client-initiated bidirectional stream limits, it
SHOULD be treated as an HTTP/3 connection error of type H3_ID_ERROR. Generating
an error is not mandatory in this case because HTTP/3 implementations might have
practical barriers to determining the active stream concurrency limit that is
applied by the QUIC layer.

HTTP/3 datagrams MUST only be sent with an association to a stream that supports
semantics for HTTP Datagrams. For example, existing HTTP methods GET and POST do
not define semantics for associated HTTP Datagrams; therefore, HTTP/3 datagrams
cannot be sent associated with GET or POST request streams. If an endpoint
receives an HTTP/3 datagram associated with a method that has no known semantics
for HTTP Datagrams, it MUST abort the corresponding stream with
H3_DATAGRAM_ERROR. Future extensions MAY remove these requirements if they
define semantics for such HTTP Datagrams and negotiate mutual support.


## The H3_DATAGRAM HTTP/3 SETTINGS Parameter {#setting}

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

It is RECOMMENDED that implementations that support receiving HTTP Datagrams
using QUIC always send the H3_DATAGRAM SETTINGS parameter with a value of 1,
even if the application does not intend to use HTTP Datagrams. This helps to
avoid "sticking out"; see {{security}}.


### Note About Draft Versions

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


# Capsules {#capsule}

This specification introduces the Capsule Protocol. The Capsule Protocol is a
sequence of type-length-value tuples that new HTTP Upgrade Tokens (see {{Section
16.7 of !HTTP=I-D.ietf-httpbis-semantics}}) can choose to use. It allows
endpoints to reliably communicate request-related information end-to-end on HTTP
request streams, even in the presence of HTTP intermediaries. The Capsule
Protocol can be used to exchange HTTP Datagrams when HTTP is running over a
transport that does not support the QUIC DATAGRAM frame.

This specification defines the "data stream" of an HTTP request as the
bidirectional stream of bytes that follow the headers in both directions. In
HTTP/1.x, the data stream consists of all bytes on the connection that follow
the blank line that concludes either the request header section, or the 2xx
(Successful) response header section. (Note that only a single HTTP request
starting the capsule protocol can be sent on HTTP/1.x connections.) In HTTP/2
and HTTP/3, the data stream of a given HTTP request consists of all bytes sent
in DATA frames with the corresponding stream ID. The concept of a data stream is
particularly relevant for methods such as CONNECT where there is no HTTP message
content after the headers.

Note that use of the Capsule Protocol is not required to use HTTP Datagrams. If
a new HTTP Upgrade Token is only defined over transports that support QUIC
DATAGRAM frames, they might not need a stream encoding. Additionally,
definitions of new HTTP Upgrade Tokens can use HTTP Datagrams with their own
data stream protocol. However, new HTTP Upgrade Tokens that wish to use HTTP
Datagrams SHOULD use the Capsule Protocol unless they have a good reason not to.


## Capsule Protocol {#capsule-protocol}

Definitions of new HTTP Upgrade Tokens can state that their data stream uses the
Capsule Protocol. If they do so, that means that the contents of their data
stream uses the following format (using the notation from the "Notational
Conventions" section of {{QUIC}}):

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

Because new protocols or extensions may involve defining new capsule types,
intermediaries that wish to allow for future extensibility SHOULD forward
capsules unmodified. One exception to this rule is the DATAGRAM capsule; see
{{datagram-capsule}}. An intermediary can identify the use of the capsule
protocol either through the presence of the Capsule-Protocol header field
({{hdr}}) or by understanding the chosen HTTP Upgrade token. An intermediary
that identifies the use of the capsule protocol MAY convert between DATAGRAM
capsules and QUIC DATAGRAM frames when forwarding. Definitions of new Capsule
Types MAY specify optional custom intermediary processing.

Endpoints which receive a Capsule with an unknown Capsule Type MUST silently
drop that Capsule.

By virtue of the definition of the data stream, the Capsule Protocol is not in
use on responses unless the response includes a 2xx (Successful) status code.

The Capsule Protocol MUST NOT be used with messages that contain Content-Length,
Content-Type, or Transfer-Encoding header fields. Additionally, HTTP status
codes 204 (No Content), 205 (Reset Content), and 206 (Partial Content) MUST NOT
be sent on responses that use the Capsule Protocol.


## Error Handling

When an error occurs processing the capsule protocol, the receiver MUST treat
the message as malformed or incomplete, according to the underlying transport
protocol.  For HTTP/3, the handling of malformed messages is described in
{{Section 4.1.3 of H3}}.  For HTTP/2, the handling of malformed messages is
described in {{Section 8.1.1 of !H2=I-D.draft-ietf-httpbis-http2bis}}.  For
HTTP/1.1, the handling of incomplete messages is described in {{Section 8 of
!H1=I-D.draft-ietf-httpbis-messaging}}.

Each capsule's payload MUST contain exactly the fields identified in its
description. A capsule payload that contains additional bytes after the
identified fields or a capsule payload that terminates before the end of the
identified fields MUST be treated as a malformed or incomplete message. In
particular, redundant length encodings MUST be verified to be self-consistent.

When a stream carrying capsules terminates cleanly, if the last capsule on the
stream was truncated, this MUST be treated as a malformed or incomplete message.


## The Capsule-Protocol Header Field {#hdr}

This document defines the "Capsule-Protocol" header field. It is an Item
Structured Field, see {{Section 3.3 of !STRUCT-FIELD=RFC8941}}; its value MUST
be a Boolean. Its ABNF is:

~~~ abnf
Capsule-Protocol = sf-item
~~~

Endpoints indicate that the Capsule Protocol is in use on the data stream by
sending the Capsule-Protocol header field with a value of ?1. A Capsule-Protocol
header field with a value of ?0 has the same semantics as when the header is not
present. Intermediaries MAY use this header field to allow processing of HTTP
Datagrams for unknown HTTP Upgrade Tokens; note that this is only possible for
HTTP Upgrade or Extended CONNECT.

The Capsule-Protocol header field MUST NOT be sent multiple times on a message.
The Capsule-Protocol header field MUST NOT be used on HTTP responses with a
status code different from 2xx (Successful). This specification does not define
any parameters for the Capsule-Protocol header field value, but future documents
MAY define parameters. Receivers MUST ignore unknown parameters.

Definitions of new HTTP Upgrade Tokens that use the Capsule Protocol MAY use the
Capsule-Protocol header field to simplify intermediary processing.


## The DATAGRAM Capsule {#datagram-capsule}

This document defines the DATAGRAM capsule type (see {{iana-types}} for the
value of the capsule type). This capsule allows an endpoint to send an HTTP
Datagram on a stream using the Capsule Protocol. This is particularly useful
when HTTP is running over a transport that does not support the QUIC DATAGRAM
frame.

~~~
Datagram Capsule {
  Type (i) = DATAGRAM,
  Length (i),
  HTTP Datagram Payload (..),
}
~~~
{: #datagram-capsule-format title="DATAGRAM Capsule Format"}

HTTP Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Datagrams sent using the DATAGRAM capsule have the same semantics as datagrams
sent in QUIC DATAGRAM frames. In particular, the restrictions on when it is
allowed to send an HTTP Datagram and how to process them from {{format}} also
apply to HTTP Datagrams sent and received using the DATAGRAM capsule.

An intermediary can reencode HTTP Datagrams as it forwards them. In other words,
an intermediary MAY send a DATAGRAM capsule to forward an HTTP Datagram which
was received in a QUIC DATAGRAM frame, and vice versa.

Note that while DATAGRAM capsules that are sent on a stream are reliably
delivered in order, intermediaries can reencode DATAGRAM capsules into QUIC
DATAGRAM frames when forwarding messages, which could result in loss or
reordering.

If an intermediary receives an HTTP Datagram in a QUIC DATAGRAM frame and is
forwarding it on a connection that supports QUIC DATAGRAM frames, the
intermediary SHOULD NOT convert that HTTP Datagram to a DATAGRAM capsule. If the
HTTP Datagram is too large to fit in a DATAGRAM frame (for example because the
path MTU of that QUIC connection is too low or if the maximum UDP payload size
advertised on that connection is too low), the intermediary SHOULD drop the HTTP
Datagram instead of converting it to a DATAGRAM capsule. This preserves the
end-to-end unreliability characteristic that methods such as Datagram
Packetization Layer Path MTU Discovery (DPLPMTUD) depend on
{{?DPLPMTUD=RFC8899}}. An intermediary that converts QUIC DATAGRAM frames to
DATAGRAM capsules allows HTTP Datagrams to be arbitrarily large without
suffering any loss; this can misrepresent the true path properties, defeating
methods such as DPLPMTUD.

While DATAGRAM capsules can theoretically carry a payload of length
2<sup>62</sup>-1, most applications will have their own limits on what datagran
payload sizes are practical. Implementations SHOULD take those limits into
account when parsing DATAGRAM capsules: if an incoming DATAGRAM capsule has a
length that is known to be so large as to not be usable, the implementation
SHOULD discard the capsule without buffering its contents into memory.


# Prioritization

Data streams (see {{capsule-protocol}}) can be prioritized using any means
suited to stream or request prioritization. For example, see {{Section 11 of
?PRIORITY=I-D.ietf-httpbis-priority}}.

Prioritization of HTTP/3 datagrams is not defined in this document. Future
extensions MAY define how to prioritize datagrams, and MAY define signaling to
allow endpoints to communicate their prioritization preferences.


# Security Considerations {#security}

Since transmitting HTTP Datagrams using QUIC DATAGRAM frames requires sending an
HTTP/3 Settings parameter, it "sticks out". In other words, probing clients can
learn whether a server supports HTTP Datagrams over QUIC DATAGRAM frames. As
some servers might wish to obfuscate the fact that they offer application
services that use HTTP datagrams, it's best for all implementations that support
this feature to always send this Settings parameter, see {{setting}}.

Since use of the Capsule Protocol is restricted to new HTTP Upgrade Tokens, it
is not accessible from Web Platform APIs (such as those commonly accessed via
JavaScript in web browsers).


# IANA Considerations {#iana}

## HTTP/3 SETTINGS Parameter {#iana-setting}

This document will request IANA to register the following entry in the
"HTTP/3 Settings" registry:

Value:

: 0xffd277 (note that this will switch to a lower value before publication)

Setting Name:

: H3_DATAGRAM

Default:

: 0

Status:

: provisional (permanent if this document is approved)

Specification:

: This Document

Change Controller:

: IETF

Contact:

: HTTP_WG; HTTP working group; ietf-http-wg@w3.org


## HTTP/3 Error Code {#iana-error}

This document will request IANA to register the following entry in the
"HTTP/3 Error Codes" registry:

Value:

: 0x4A1268 (note that this will switch to a lower value before publication)

Name:

: H3_DATAGRAM_ERROR

Description:

: Datagram or capsule protocol parse error

Status:

: provisional (permanent if this document is approved)

Specification:

: This Document

Change Controller:

: IETF

Contact:

: HTTP_WG; HTTP working group; ietf-http-wg@w3.org


## HTTP Header Field Name {#iana-hdr}

This document will request IANA to register the following entry in the "HTTP
Field Name" registry:

Field Name:

: Capsule-Protocol

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

This registry initially contains the following entry:

| Capsule Type                 |   Value   | Specification |
|:-----------------------------|:----------|:--------------|
| DATAGRAM                     | 0xff37a5  | This Document |
{: #iana-types-table title="Initial Capsule Types Registry"}

Capsule types with a value of the form 41 * N + 23 for integer values of N are
reserved to exercise the requirement that unknown capsule types be ignored.
These capsules have no semantics and can carry arbitrary values. These values
MUST NOT be assigned by IANA and MUST NOT appear in the listing of assigned
values.


--- back

# Examples

\[\[RFC editor: please remove this appendix before publication.]]

## CONNECT-UDP

~~~
Client                                             Server

STREAM(44): HEADERS             -------->
  :method = CONNECT
  :protocol = connect-udp
  :scheme = https
  :path = /target.example.org/443/
  :authority = proxy.example.org:443
  capsule-protocol = ?1

DATAGRAM                        -------->
  Quarter Stream ID = 11
  Payload = Encapsulated UDP Payload

           <--------  STREAM(44): HEADERS
                        :status = 200
                        capsule-protocol = ?1

/* Wait for target server to respond to UDP packet. */

           <--------  DATAGRAM
                        Quarter Stream ID = 11
                        Payload = Encapsulated UDP Payload
~~~


## WebTransport

~~~
Client                                             Server

STREAM(44): HEADERS            -------->
  :method = CONNECT
  :scheme = https
  :protocol = webtransport
  :path = /hello
  :authority = webtransport.example.org:443
  origin = https://www.example.org:443

           <--------  STREAM(44): HEADERS
                        :status = 200

/* Both endpoints can now send WebTransport datagrams. */
~~~


# Acknowledgments {#acks}
{:numbered="false"}

The DATAGRAM context identifier was previously part of the DATAGRAM frame
definition itself, the authors would like to acknowledge the authors of that
document and the members of the IETF MASQUE working group for their suggestions.
Additionally, the authors would like to thank Martin Thomson for suggesting the
use of an HTTP/3 SETTINGS parameter. Furthermore, the authors would like to
thank Ben Schwartz for writing the first proposal that used two layers of
indirection.
