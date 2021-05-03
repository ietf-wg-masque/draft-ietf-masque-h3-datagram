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
HTTP/3. It defines logical flows identified by a non-negative integer that are
present at the start of the DATAGRAM frame payload. Flows are associated with
HTTP messages using the Datagram-Flow-Id header field, allowing endpoints to
match unreliable DATAGRAMS frames to the HTTP messages that they are related to.

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
HTTP/3 {{!H3=I-D.ietf-quic-http}}. It defines logical flows identified by a
non-negative integer that are present at the start of the DATAGRAM frame
payload. Flows are associated with HTTP messages using the Datagram-Flow-Id
header field, allowing endpoints to match unreliable DATAGRAMS frames to the
HTTP messages that they are related to.

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


# Datagram Flows

Flows are bidirectional exchanges of datagrams within a single QUIC connection.
These are conceptually similar to streams in the sense that they allow
multiplexing of application data. Flows are identified within a connection by a
numeric value, referred to as the flow ID.  A flow ID is a 62-bit integer (0
to 2^62-1).

Flows lack any of the ordering or reliability guarantees of streams. Beyond
this, a sender SHOULD ensure that DATAGRAM frames within a single flow are
transmitted in order relative to one another.


# Flow ID Allocation {#flow-id-alloc}

Implementations of HTTP/3 that support the DATAGRAM extension MUST provide a
flow ID allocation service. That service will allow applications
co-located with HTTP/3 to request a unique flow ID that they can
subsequently use for their own purposes. The HTTP/3 implementation will then
parse the flow ID of incoming DATAGRAM frames and use it to deliver the
frame to the appropriate application context.

Even-numbered flow IDs are client-initiated, while odd-numbered flow
IDs are server-initiated. This means that an HTTP/3 client
implementation of the flow ID allocation service MUST only provide
even-numbered IDs, while a server implementation MUST only provide
odd-numbered IDs. Note that, once allocated, any flow ID can be
used by both client and server - only allocation carries separate namespaces to
avoid requiring synchronization.


# HTTP/3 DATAGRAM Frame Format {#format}

When used with HTTP/3, the Datagram Data field of QUIC DATAGRAM frames uses the
following format (using the notation from the "Notational Conventions" section
of {{QUIC}}):

~~~
HTTP/3 DATAGRAM Frame {
  Flow ID (i),
  HTTP/3 Datagram Payload (..),
}
~~~
{: #h3-datagram-format title="HTTP/3 DATAGRAM Frame Format"}

Flow ID:

: A variable-length integer indicating the Flow ID of the datagram (see
{{datagram-flows}}).

HTTP/3 Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Endpoints MUST treat receipt of a DATAGRAM frame whose payload is too short to
parse the flow ID as an HTTP/3 connection error of type
H3_GENERAL_PROTOCOL_ERROR.


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


# Datagram-Flow-Id Header Field Definition {#header}

"Datagram-Flow-Id" is a List Structured
Field {{!STRUCT-FIELD=I-D.ietf-httpbis-header-structure}}, whose members MUST
all be Items of type Integer. Integers MUST be non-negative. Its ABNF is:

~~~
  Datagram-Flow-Id = sf-list
~~~

The "Datagram-Flow-Id" header field is used to associate one or more datagram
flows with an HTTP message. As a simple example using a single
flow, the definition of an HTTP method could instruct the client to use
its flow ID allocation service to allocate a new flow ID, and
then the client will add the "Datagram-Flow-Id" header field to its request to
communicate that value to the server. In this example, the resulting header
field could look like:

~~~
  Datagram-Flow-Id = 2
~~~

List members are flow ID elements, which can be named or unnamed.
One element in the list is allowed to be unnamed, but all but one elements
MUST carry a name. The name of an element is encoded in the key of the first
parameter of that element (parameters are defined in Section 3.1.2 of
{{STRUCT-FIELD}}). Each name MUST NOT appear more than once in the list. The
value of the first parameter of each named element (whose corresponding key
conveys the element name) MUST be of type Boolean and equal to true. The value
of the first parameter of the unnamed element MUST NOT be of type Boolean. The
ordering of the list does not carry any semantics. For example, an HTTP method
that wishes to use four datagram flows for the lifetime of its
request stream could look like this:

~~~
  Datagram-Flow-Id = 42, 44; foo, 46; bar, 48; baz
~~~

In this example, 42 is the unnamed flow, 44 represents the name
"foo", 46 represents "bar", and 48 represents "baz". Note that, since the list
ordering does not carry semantics, this example can be equivalently encoded as:

~~~
  Datagram-Flow-Id = 44; foo, 42, 48; baz, 46; bar
~~~

Even if a sender attempts to communicate the meaning of a flow
before it uses it in an HTTP/3 datagram, it is possible that its peer will
receive an HTTP/3 datagram with a flow ID that it does not know as it
has not yet received the corresponding "Datagram-Flow-Id" header field. (For
example, this could happen if the QUIC STREAM frame that contains the
"Datagram-Flow-Id" header field is reordered and arrives afer the DATAGRAM
frame.) Endpoints MUST NOT treat that scenario as an error; they MUST either
silently discard the datagram or buffer it until they receive the
"Datagram-Flow-Id" header field.

Distinct HTTP requests MAY refer to the same flow in their
respective "Datagram-Flow-Id" header fields.

Note that integer structured fields can only encode values up to 10^15-1,
therefore the maximum possible value of an element of the "Datagram-Flow-Id"
header field is lower then the theoretical maximum value of a flow ID
which is 2^62-1 due to the QUIC variable length integer encoding. If the flow
allocation service of an endpoint runs out of flow ID values lower than
10^15-1, the endpoint MUST fail the flow ID allocation. An HTTP
message that carries a "Datagram-Flow-Id" header field with a flow ID
value above 10^15-1 is malformed (see Section 8.1.2.6 of {{!H2=RFC7540}}).


# HTTP Intermediaries {#intermediaries}

HTTP/3 DATAGRAM flows are specific to a given HTTP/3 connection.
However, in some cases, an HTTP request may travel across multiple HTTP
connections if there are HTTP intermediaries involved; see Section 2.3 of
{{!RFC7230}}.

If an intermediary has sent the H3_DATAGRAM SETTINGS parameter with a value of 1
on its client-facing connection, it MUST inspect all HTTP requests from that
connection and check for the presence of the "Datagram-Flow-Id" header field. If
the HTTP method of the request is not supported by the intermediary, it MUST
remove the "Datagram-Flow-Id" header field before forwarding the request. If the
intermediary supports the method, it MUST either remove the header field or
adhere to the requirements leveraged by that method on intermediaries.

If an intermediary has sent the H3_DATAGRAM SETTINGS parameter with a value of 1
on its server-facing connection, it MUST inspect all HTTP responses from that
connection and check for the presence of the "Datagram-Flow-Id" header field. If
the HTTP method of the request is not supported by the intermediary, it MUST
remove the "Datagram-Flow-Id" header field before forwarding the response. If
the intermediary supports the method, it MUST either remove the header field or
adhere to the requirements leveraged by that method on intermediaries.

If an intermediary processes distinct HTTP requests that refer to the same flow
ID in their respective "Datagram-Flow-Id" header fields, it MUST ensure
that those requests are routed to the same backend.


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


## HTTP Header Field {#iana-header}

This document will request IANA to register the "Datagram-Flow-Id"
header field in the "Permanent Message Header Field Names"
registry maintained at
<[](https://www.iana.org/assignments/message-headers)>.

~~~
  +-------------------+----------+--------+---------------+
  | Header Field Name | Protocol | Status |   Reference   |
  +-------------------+----------+--------+---------------+
  | Datagram-Flow-Id  |   http   |  std   | This document |
  +-------------------+----------+--------+---------------+
~~~


## Flow Parameters {#iana-params}

This document will request IANA to create an "HTTP Datagram Flow
Parameters" registry. Registrations in this registry MUST
include the following fields:

Key:

: The key of a parameter that is associated with a datagram flow
list member (see {{header}}). Keys MUST be valid structured field parameter
keys (see Section 3.1.2 of {{STRUCT-FIELD}}).

Description:

: A brief description of the parameter semantics, which MAY be a summary if a
specification reference is provided.

Is Name:

: This field MUST be either Yes or No. Yes indicates that this
parameter is the name of a named element (see {{header}}). No indicates that it
is a parameter that is not a name.

Reference:

: An optional reference to a specification for the parameter. This field MAY be
empty.

Registrations follow the "First Come First Served" policy (see Section 4.4 of
{{!IANA-POLICY=RFC8126}}) where two registrations MUST NOT have the same Key.
This registry is initially empty.


--- back

# Acknowledgments {#acks}
{:numbered="false"}

The DATAGRAM flow identifier was previously part of the DATAGRAM frame
definition itself, the author would like to acknowledge the authors of
that document and the members of the IETF MASQUE working group for their
suggestions. Additionally, the author would like to thank Martin Thomson
for suggesting the use of an HTTP/3 SETTINGS parameter.
