---
title: HTTP Datagrams and the Capsule Protocol
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

normative:
  QUIC: RFC9000
  H3: I-D.ietf-quic-http
  H2: I-D.ietf-httpbis-http2bis


--- abstract

This document describes HTTP Datagrams, a convention for conveying multiplexed,
potentially unreliable datagrams inside an HTTP connection.

In HTTP/3, HTTP Datagrams can be conveyed natively using the QUIC DATAGRAM
extension. To enable seamless transfer of HTTP Datagrams in other protocol
versions, this document also defines the Capsule Protocol, a more general
convention for conveying data in HTTP connections.

Both are intended for use by HTTP extensions, not applications.


--- middle

# Introduction {#intro}

HTTP extensions sometimes need to access underlying transport protocol features
such as unreliable delivery (as offered by {{!DGRAM=I-D.ietf-quic-datagram}})
to enable desirable features like an unreliable version of the CONNECT method,
and unreliable delivery in WebSockets {{?RFC6455}} (or its successors).

In {{datagrams}}, this document describes HTTP Datagrams, a convention for
conveying multiplexed, bidirectional datagrams inside a HTTP connection. They
are intended for use by HTTP extensions (such as the CONNECT method), and are
compatible with all versions of HTTP. When the underlying transport protocol
supports unreliable delivery (such as when the QUIC DATAGRAM extension is
available in HTTP/3), they use that capability.

It also describes the HTTP Capsule Protocol in {{capsule}}, to allow conveyance
of HTTP Datagrams over other versions of HTTP.


## Conventions and Definitions {#defs}

{::boilerplate bcp14-tagged}


# HTTP Datagrams {#datagrams}

HTTP Datagrams are a convention for conveying bidirectional and potentially
unreliable datagrams inside an HTTP connection, with multiplexing when
possible. All HTTP Datagrams are associated with an HTTP request.

When HTTP Datagrams are conveyed on a HTTP/3 connection, the QUIC DATAGRAM
frame is used to achieve these goals, including unreliable delivery; see
{{format}}. Negotiation is achieved using a SETTING; see {{setting}}.

When running over HTTP/2, demultiplexing is provided by the HTTP/2 framing
layer, but unreliable delivery is unavailable. HTTP Datagrams are negotiated
and conveyed using the Capsule Protocol; see {{datagram-capsule}}.

When running over HTTP/1, requests are strictly serialized in the connection,
and therefore demultiplexing is not available. Unreliable delivery is likewise
not available. HTTP Datagrams are negotiated and conveyed using the Capsule
Protocol; see {{datagram-capsule}}.


## HTTP/3 Datagrams {#format}

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

: The payload of the datagram, whose semantics are defined by the
application. Note that this field can be empty.

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
silently or buffer it temporarily (on the order of a round trip) while awaiting
the creation of the corresponding stream.

If an HTTP/3 datagram is received and its Quarter Stream ID maps to a stream
that cannot be created due to client-initiated bidirectional stream limits, it
SHOULD be treated as an HTTP/3 connection error of type H3_ID_ERROR. Generating
an error is not mandatory in this case because HTTP/3 implementations might have
practical barriers to determining the active stream concurrency limit that is
applied by the QUIC layer.

HTTP/3 datagrams MUST only be sent with an association to a stream whose HTTP
semantics explicitly supports HTTP Datagrams. For example, existing HTTP
methods GET and POST do not define semantics for associated HTTP Datagrams;
therefore, HTTP/3 datagrams cannot be sent associated with GET or POST request
streams.

If an endpoint receives an HTTP/3 datagram associated with a method
that has no known semantics for HTTP Datagrams, it MUST abort the corresponding
stream with H3_DATAGRAM_ERROR. HTTP extensions can override these requirements
by defining a negotiation mechanism and semantics for HTTP Datagrams.

Prioritization of HTTP/3 datagrams is not defined in this document. Future
extensions MAY define how to prioritize datagrams, and MAY define signaling to
allow endpoints to communicate their prioritization preferences.


### The H3_DATAGRAM HTTP/3 SETTINGS Parameter {#setting}

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


#### Note About Draft Versions

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



## The DATAGRAM Capsule {#datagram-capsule}

The DATAGRAM Capsule Type conveys a HTTP Datagram using the Capsule Protocol (see {{capsule}}), primarily for the purpose of using HTTP Datagrams on versions of the protocol that do not support the HTTP/3 Datagram Format (see {{format}}).

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

HTTP Datagrams sent using the DATAGRAM capsule have the same semantics as those
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
2<sup>62</sup>-1, most applications will have their own limits on what datagram
payload sizes are practical. Implementations SHOULD take those limits into
account when parsing DATAGRAM capsules: if an incoming DATAGRAM capsule has a
length that is known to be so large as to not be usable, the implementation
SHOULD discard the capsule without buffering its contents into memory.


# The Capsule Protocol {#capsule}

The Capsule Protocol conveys a sequence of type-length-value tuples on a HTTP
data stream (see {{data-stream}}), allowing endpoints to communicate end-to-end
even in the presence of HTTP intermediaries. Its use can be negotiated through
the HTTP Upgrade mechanism (see {{Section 16.7 of
!HTTP=I-D.ietf-httpbis-semantics}}), or it can be invoked by an extension to
the CONNECT method (for example, see {{Section 8.5 of
H2}}).

Intermediaries can identify when the Capsule Protocol is in use by examining the Capsule-Protocol header field ({{hdr}}) and the HTTP Upgrade token.

Because new protocols or extensions might define new capsule types,
intermediaries that wish to allow for future extensibility SHOULD forward
capsules without modification, unless the definition of the Capsule Type in use
specifies additional intermediary processing. One such Capsule Type is the
DATAGRAM capsule; see {{datagram-capsule}}.

An intermediary MAY convert between DATAGRAM capsules and QUIC DATAGRAM frames
when forwarding them.

Endpoints which receive a capsule with an unknown Capsule Type MUST silently
drop that Capsule.

Per the definition of the data stream, the Capsule Protocol is not in
use on a response that does not have a 2xx (Successful) status code.

The Capsule Protocol MUST NOT be used with messages that contain Content-Length,
Content-Type, or Transfer-Encoding header fields. Additionally, HTTP status
codes 204 (No Content), 205 (Reset Content), and 206 (Partial Content) MUST NOT
be sent on responses that use the Capsule Protocol.

Data streams using the Capsule Protocol can be prioritized using any means
suited to stream or request prioritization. For example, see {{Section 11 of
?PRIORITY=I-D.ietf-httpbis-priority}}.


## HTTP Data Streams {#data-stream}

In the Capsule Protocol, the "data stream" of an HTTP request is the
bidirectional stream of bytes that follows the header section of the request
message and the final, successful (i.e., 2xx) response message.

In HTTP/1.x, the data stream consists of all bytes on the connection that
follow the blank line that concludes either the request header section, or the
s response header section. As a result, only a single HTTP
request starting the capsule protocol can be sent on HTTP/1.x connections.

In HTTP/2 and HTTP/3, the data stream of a given HTTP request consists of all
bytes sent in DATA frames with the corresponding stream ID.

The concept of a data stream is particularly relevant for methods such as
CONNECT where there is no HTTP message content after the headers.

Note that use of the Capsule Protocol is not required to use HTTP Datagrams. If
a new HTTP Upgrade Token is only defined over transports that support QUIC
DATAGRAM frames, they might not need a stream encoding. Additionally,
definitions of new HTTP Upgrade Tokens can use HTTP Datagrams with their own
data stream protocol. However, new HTTP Upgrade Tokens that wish to use HTTP
Datagrams SHOULD use the Capsule Protocol unless they have a good reason not to.


## Capsule Format {#capsule-protocol}

When a data stream uses the Capsule Protocol, data is conveyed uses this format
(specified using the notation from the "Notational Conventions" section of
{{QUIC}}):

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

: A variable-length integer indicating the Type of the capsule. Recipients MUST silently skip over a capsule with an unknown Capsule Type.

Capsule Length:

: The length of the Capsule Value field following this field, encoded as a
variable-length integer. Note that this field can have a value of zero.

Capsule Value:

: The payload of this capsule. Its semantics are determined by the value of the
Capsule Type field.


## Error Handling

When an error occurs in processing the capsule protocol, the receiver MUST treat
the message as malformed or incomplete, according to the underlying transport
protocol.  For HTTP/3, the handling of malformed messages is described in
{{Section 4.1.3 of H3}}.  For HTTP/2, the handling of malformed messages is
described in {{Section 8.1.1 of H2}}.  For
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

The "Capsule-Protocol" header field is an Item Structured Field, see {{Section
3.3 of !STRUCT-FIELD=RFC8941}}; its value MUST be a Boolean.

Endpoints indicate that the Capsule Protocol is in use on a data stream by
sending a Capsule-Protocol header field with a value of ?1. A Capsule-Protocol
header field with a value of ?0 has the same semantics as when the header is not
present.

Intermediaries MAY use this header field to allow processing of HTTP Datagrams
for unknown HTTP Upgrade Tokens; note that this is only possible for HTTP
Upgrade or Extended CONNECT.

The Capsule-Protocol header field MUST NOT be used on HTTP responses with a
status code different from 2xx (Successful). This specification does not define
any parameters for the Capsule-Protocol header field value, but future documents
MAY define parameters. Receivers MUST ignore unknown parameters.

Definitions of new HTTP Upgrade Tokens that use the Capsule Protocol MAY use the
Capsule-Protocol header field to simplify intermediary processing.


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

Capsule Type:

: DATAGRAM

Value:

: 0xff37a5 (note that this will switch to a lower value before publication)

Reference:

: This document

Capsule types with a value of the form 41 * N + 23 for integer values of N are
reserved to exercise the requirement that unknown capsule types be ignored.
These capsules have no semantics and can carry arbitrary values. These values
MUST NOT be assigned by IANA and MUST NOT appear in the listing of assigned
values.


--- back

# Acknowledgments {#acks}
{:numbered="false"}

Portions of this document were previously part of the QUIC DATAGRAM frame
definition itself, the authors would like to acknowledge the authors of that
document and the members of the IETF MASQUE working group for their suggestions.
Additionally, the authors would like to thank Martin Thomson for suggesting the
use of an HTTP/3 SETTINGS parameter. Furthermore, the authors would like to
thank Ben Schwartz for writing the first proposal that used two layers of
indirection. The final design in this document came out of the HTTP Datagrams
Design Team, whose members were Alan Frindell, Alex Chernyakhovsky, Ben
Schwartz, Eric Rescorla, Marcus Ihlar, Martin Thomson, Mike Bishop, Tommy Pauly,
Victor Vasiliev, and the authors of this document.
