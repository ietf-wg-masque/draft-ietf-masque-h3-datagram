---
title: Using QUIC Datagrams with HTTP/3
abbrev: HTTP/3 Datagrams
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


--- abstract

The QUIC DATAGRAM extension provides application protocols running over QUIC
with a mechanism to send unreliable data while leveraging the security and
congestion-control properties of QUIC. However, QUIC DATAGRAM frames do not
provide a means to demultiplex application contexts. This document describes how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3. It associates datagrams with client-initiated bidirectional streams
and defines an optional additional demultiplexing layer.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing list
([masque@ietf.org](mailto:masque@ietf.org)) or on the GitHub repository which
contains the draft:
[](https://github.com/ietf-wg-masque/draft-ietf-masque-h3-datagram).


--- middle

# Introduction {#intro}

The QUIC DATAGRAM extension {{!DGRAM=I-D.ietf-quic-datagram}} provides
application protocols running over QUIC {{!QUIC=I-D.ietf-quic-transport}} with a
mechanism to send unreliable data while leveraging the security and
congestion-control properties of QUIC. However, QUIC DATAGRAM frames do not
provide a means to demultiplex application contexts. This document describes how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3 {{!H3=I-D.ietf-quic-http}}. It associates datagrams with client-initiated
bidirectional streams and defines an optional additional demultiplexing layer.

This design mimics the use of Stream Types in HTTP/3, which provide a
demultiplexing identifier at the start of each unidirectional stream.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing list
([masque@ietf.org](mailto:masque@ietf.org)) or on the GitHub repository which
contains the draft:
[](https://github.com/ietf-wg-masque/draft-ietf-masque-h3-datagram).


## Conventions and Definitions {#defs}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Multiplexing

In order to allow multiple datagram types and contexts to coexist on a given
QUIC connection, HTTP datagrams contain two layers of multiplexing. First, the
QUIC DATAGRAM frame payload starts with a custom-encoded stream identifier that
associates the datagram with a given QUIC stream. Second, datagrams carry a
flow ID that allows multiplexing multiple datagram flows related to a given
HTTP request. Conceptually, the first layer of multiplexing is per-hop, while
the second is end-to-end (see {{datagram-flows}}).


# Datagram Flows {#datagram-flows}

Flows are unirectional exchanges of datagrams associated with a given HTTP
request. Flows are identified within a request context by a numeric value,
referred to as the flow ID. A flow ID is a 62-bit integer (0 to 2^62-1).

While stream IDs are a per-hop concept, flow IDs are an end-to-end concept. In
other words, if a datagram travels through multiple intermediaries on its way
from client to server, the stream ID will most likely change from hop to hop,
but the flow ID will remain the same. Flow IDs are opaque to intermediaries.


## Flow ID Allocation {#flow-id-alloc}

Implementations of HTTP/3 that support the DATAGRAM extension MUST provide a
flow ID allocation service. That service will allow applications
co-located with HTTP/3 to request a unique flow ID that they can
subsequently use for their own purposes. The HTTP/3 implementation will then
parse the flow ID of incoming DATAGRAM frames and use it to deliver the
frame to the appropriate application context. Note that the flow ID namespace is
tied to a given HTTP request: it is possible for the same numeral flow ID to be
used simultaneously in distinct requests.


# HTTP/3 DATAGRAM Frame Format {#format}

When used with HTTP/3, the Datagram Data field of QUIC DATAGRAM frames uses the
following format (using the notation from the "Notational Conventions" section
of {{QUIC}}):

~~~
HTTP/3 DATAGRAM Frame {
  Quarter Stream ID (i),
  Flow ID (i),
  HTTP/3 Datagram Payload (..),
}
~~~
{: #h3-datagram-format title="HTTP/3 DATAGRAM Frame Format"}

Quarter Stream ID:

: A variable-length integer that contains the value of the client-initiated
bidirectional stream that this datagram is associated with, divided by four.
(The division by four stems from the fact that HTTP requests are sent on
client-initiated bidirectional streams, and those have stream IDs equal to zero
modulo four.)

Flow ID:

: A variable-length integer indicating the Flow ID of the datagram (see
{{datagram-flows}}).

HTTP/3 Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Endpoints MUST treat receipt of a DATAGRAM frame whose payload is too short to
parse the Quarter Stream ID or Flow ID as an HTTP/3 connection error of type
H3_GENERAL_PROTOCOL_ERROR.


# REGISTER_DATAGRAM_FLOW_ID HTTP/3 Frame Definition {#register-frame}

The REGISTER_DATAGRAM_FLOW_ID frame (type=TBD) allows an endpoint to inform its
peer of the encoding and semantics of upcoming datagrams associated with this
stream.

~~~
REGISTER_DATAGRAM_FLOW_ID Frame {
  Type (i) = TBD,
  Length (i),
  Flow ID (i),
  Extension String (..),
}
~~~
{: #register-frame-format title="REGISTER_DATAGRAM_FLOW_ID HTTP/3 Frame Format"}

The Type and Length fields follows the definition of HTTP/3 frames from {{H3}}.
The payload consists of:

Flow ID:

: The flow ID to associate with the QUIC stream that carried this frame.

Extension String:

: A string of comma-separated key-value pairs to enable extensibility.

The ABNF for the Extension String field is as follows (using syntax from
{{Section 3.2.6 of RFC7230}}):

~~~
  extension-string = [ ext-member *( "," ext-member ) ]
  ext-member       = ext-member-key "=" ext-member-value
  ext-member-key   = token
  ext-member-value = token
~~~

Note that these registrations are unilateral and unidirectional: the sender of
the frame unilateraly defines the semantics it will apply to the datagrams it
sends. If a mechanism using this feature wants to send datagrams of a given
flow ID in both directions, this frame will need to be exchanged in both
directions.


# The H3_DATAGRAM HTTP/3 SETTINGS Parameter {#setting}

Implementations of HTTP/3 that support this mechanism can indicate that to
their peer by sending the H3_DATAGRAM SETTINGS parameter with a value of 1.
The value of the H3_DATAGRAM SETTINGS parameter MUST be either 0 or 1. A value
of 0 indicates that this mechanism is not supported. An endpoint that receives
the H3_DATAGRAM SETTINGS parameter with a value that is neither 0 or 1 MUST
terminate the connection with error H3_SETTINGS_ERROR.

An endpoint that sends the H3_DATAGRAM SETTINGS parameter with a value of 1
MUST send the max_datagram_frame_size QUIC Transport Parameter {{DGRAM}}.
An endpoint that receives the H3_DATAGRAM SETTINGS parameter with a value of 1
on a QUIC connection that did not also receive the max_datagram_frame_size
QUIC Transport Parameter MUST terminate the connection with error
H3_SETTINGS_ERROR.

When clients use 0-RTT, they MAY store the value of the server's H3_DATAGRAM
SETTINGS parameter.  Doing so allows the client to use HTTP/3 datagrams in 0-RTT
packets.  When servers decide to accept 0-RTT data, they MUST send a H3_DATAGRAM
SETTINGS parameter greater than or equal to the value they sent to the client in
the connection where they sent them the NewSessionTicket message.  If a client
stores the value of the H3_DATAGRAM SETTINGS parameter with their 0-RTT state,
they MUST validate that the new value of the H3_DATAGRAM SETTINGS parameter sent
by the server in the handshake is greater than or equal to the stored value; if
not, the client MUST terminate the connection with error H3_SETTINGS_ERROR.  In
all cases, the maximum permitted value of the H3_DATAGRAM SETTINGS parameter is
1.


# RELIABLE_DATAGRAM HTTP/3 Frame Definition {#reliable-datagram-frame}

The RELIABLE_DATAGRAM frame (type=TBD) is used to send datagrams over QUIC
streams when QUIC datagrams are unavailable or undesirable. Datagrams
transmitted over streams using this frame have the same semantics as datagrams
sent over the QUIC DATAGRAM frame. The RELIABLE_DATAGRAM frame does not carry
the Quarter Stream ID field because the Stream ID can be infered from the QUIC
STREAM frames that carry this HTTP/3 frame.

~~~
RELIABLE_DATAGRAM Frame {
  Type (i) = TBD,
  Length (i),
  Flow ID (i),
  HTTP/3 Datagram Payload (..),
}
~~~
{: #reliable-datagram-frame-format title="RELIABLE_DATAGRAM HTTP/3 Frame Format"}

The Type and Length fields follows the definition of HTTP/3 frames from {{H3}}.
The payload consists of:

Flow ID:

: A variable-length integer indicating the Flow ID of the datagram (see
{{datagram-flows}}).

HTTP/3 Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.


# HTTP/1.x and HTTP/2 Support

We can provide DATAGRAM support in HTTP/2 by defining the
REGISTER_DATAGRAM_FLOW_ID and RELIABLE_DATAGRAM frames frame in HTTP/2.

We can provide DATAGRAM support in HTTP/1.x by defining its data stream format
to a sequence of length-value datagrams.

TODO: Refactor this document into "HTTP Datagrams" with definitions for
HTTP/1.x, HTTP/2, and HTTP/3.


# Security Considerations {#security}

This document does not have additional security considerations beyond those
defined in {{QUIC}} and {{DGRAM}}.


# IANA Considerations {#iana}

## HTTP SETTINGS Parameter {#iana-setting}

This document will request IANA to register the following entry in the
"HTTP/3 Settings" registry:

~~~
  +--------------+-------+---------------+---------+
  | Setting Name | Value | Specification | Default |
  +==============+=======+===============+=========+
  | H3_DATAGRAM  | 0x276 | This Document |    0    |
  +--------------+-------+---------------+---------+
~~~


## Flow Extension Keys {#iana-keys}

This document will request IANA to create an "HTTP Datagram Flow
Extension Keys" registry. Registrations in this registry MUST
include the following fields:

Key:

: The key (see {{register-frame}}). Keys MUST be valid tokens as defined in
Section 3.2.6 of {{!RFC7230}}.

Description:

: A brief description of the key semantics, which MAY be a summary if a
specification reference is provided.

Reference:

: An optional reference to a specification for the parameter. This field MAY be
empty.

Registrations follow the "First Come First Served" policy (see Section 4.4 of
{{!IANA-POLICY=RFC8126}}) where two registrations MUST NOT have the same Key.
This registry is initially empty.


--- back

# Examples

## CONNECT-UDP

~~~
Client                                             Server

STREAM(44): DATA{HEADERS}      -------->
  :method = CONNECT-UDP
  :scheme = https
  :path = /
  :authority = target.example.org:443

STREAM(44): REGISTER_DATAGRAM_FLOW_ID  -------->
  Flow ID = 1
  Extension String = ""

DATAGRAM                       -------->
  Quarter Stream ID = 11
  Flow ID = 1
  Payload = Encapsulated UDP Payload

           <--------  STREAM(44): DATA{HEADERS}
                        :status = 200

           <--------  STREAM(44): REGISTER_DATAGRAM_FLOW_ID
                        Flow ID = 1
                        Extension String = ""

/* Wait for target server to respond to UDP packet. */

           <--------  DATAGRAM
                        Quarter Stream ID = 11
                        Flow ID = 1
                        Payload = Encapsulated UDP Payload
~~~


## CONNECT-UDP with ECN

~~~
Client                                             Server

STREAM(44): DATA{HEADERS}      -------->
  :method = CONNECT-UDP
  :scheme = https
  :path = /
  :authority = target.example.org:443

STREAM(44): REGISTER_DATAGRAM_FLOW_ID  -------->
  Flow ID = 1
  Extension String = ""

DATAGRAM                       -------->
  Quarter Stream ID = 11
  Flow ID = 1
  Payload = Encapsulated UDP Payload

           <--------  STREAM(44): DATA{HEADERS}
                        :status = 200

           <--------  STREAM(44): REGISTER_DATAGRAM_FLOW_ID
                        Flow ID = 1
                        Extension String = ""

/* Wait for target server to respond to UDP packet. */

           <--------  DATAGRAM
                        Quarter Stream ID = 11
                        Flow ID = 1
                        Payload = Encapsulated UDP Payload

STREAM(44): REGISTER_DATAGRAM_FLOW_ID  -------->
  Flow ID = 2
  Extension String = "ecn=ect0"

DATAGRAM                       -------->
  Quarter Stream ID = 11
  Flow ID = 2
  Payload = Encapsulated UDP Payload
~~~


## CONNECT-IP with IP compression

~~~
Client                                             Server

STREAM(44): DATA{HEADERS}      -------->
  :method = CONNECT-IP
  :scheme = https
  :path = /
  :authority = proxy.example.org:443

           <--------  STREAM(44): DATA{HEADERS}
                        :status = 200

/* Exchange CONNECT-IP configuration information. */

STREAM(44): REGISTER_DATAGRAM_FLOW_ID  -------->
  Flow ID = 1
  Extension String = ""

DATAGRAM                       -------->
  Quarter Stream ID = 11
  Flow ID = 1
  Payload = Encapsulated IP Packet

           <--------  STREAM(44): REGISTER_DATAGRAM_FLOW_ID
                        Flow ID = 1
                        Extension String = ""

/* Endpoint happily exchange encapsulated IP packets. */

DATAGRAM                       -------->
  Quarter Stream ID = 11
  Flow ID = 1
  Payload = Encapsulated IP Packet

/* After performing some analysis on traffic patterns, */
/* the client decides it wants to compress a 5-tuple. */

STREAM(44): REGISTER_DATAGRAM_FLOW_ID  -------->
  Flow ID = 2
  Extension String = "ip=192.0.2.42,port=443"

DATAGRAM                       -------->
  Quarter Stream ID = 11
  Flow ID = 2
  Payload = Compressed IP Packet
~~~


## WebTransport

~~~
Client                                             Server

STREAM(44): DATA{HEADERS}      -------->
  :method = CONNECT
  :scheme = https
  :method = webtransport
  :path = /hello
  :authority = webtransport.example.org:443
  Origin = https://www.example.org:443

STREAM(44): REGISTER_DATAGRAM_FLOW_ID  -------->
  Flow ID = 1
  Extension String = ""

           <--------  STREAM(44): DATA{HEADERS}
                        :status = 200

           <--------  STREAM(44): REGISTER_DATAGRAM_FLOW_ID
                        Flow ID = 1
                        Extension String = ""

/* Both endpoints can now send WebTransport datagrams. */
~~~


# Acknowledgments {#acks}
{:numbered="false"}

The DATAGRAM flow identifier was previously part of the DATAGRAM frame
definition itself, the author would like to acknowledge the authors of that
document and the members of the IETF MASQUE working group for their
suggestions. Additionally, the author would like to thank Martin Thomson for
suggesting the use of an HTTP/3 SETTINGS parameter. Furthermore, the authors
would like to thank Ben Schwartz for writing the first proposal that used two
layers of indirection.
