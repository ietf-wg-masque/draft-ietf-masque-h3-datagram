---
title: Using QUIC Datagrams with HTTP/3
abbrev: HTTP/3 Datagrams
docname: draft-schinazi-masque-h3-datagram-latest
category: std

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
provide a means to demultiplex application contexts. This document defines how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3 by adding an identifier at the start of the frame payload. This allows
HTTP messages to convey related information using unreliable DATAGRAM frames,
ensuring those frames are properly associated with an HTTP message.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing list
([masque@ietf.org](mailto:masque@ietf.org)) or on the GitHub repository which
contains the draft: [](https://github.com/DavidSchinazi/draft-h3-datagram).


--- middle

# Introduction {#intro}

The QUIC DATAGRAM extension {{!DGRAM=I-D.ietf-quic-datagram}} provides
application protocols running over QUIC {{!QUIC=I-D.ietf-quic-transport}} with
a mechanism to send unreliable data while leveraging the security and
congestion-control properties of QUIC. However, QUIC DATAGRAM frames do not
provide a means to demultiplex application contexts. This document defines how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3 {{!H3=I-D.ietf-quic-http}} by adding an identifier at the start of the
frame payload. This allows HTTP messages to convey related information using
unreliable DATAGRAM frames, ensuring those frames are properly associated with
an HTTP message.

This design mimics the use of Stream Types in HTTP/3, which provide a
demultiplexing identifier at the start of each unidirectional stream.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing list
([masque@ietf.org](mailto:masque@ietf.org)) or on the GitHub repository which
contains the draft: [](https://github.com/DavidSchinazi/draft-h3-datagram).


## Conventions and Definitions {#defs}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Flow Identifiers {#flow-id}

Flow identifiers represent bidirectional flows of datagrams within a single QUIC
connection. These are conceptually similar to streams in the sense that they
allow multiplexing of application data.  Flows lack any of the ordering
or reliability guarantees of streams.

Beyond this, a sender SHOULD ensure that DATAGRAM frames within a single flow
are transmitted in order relative to one another. If multiple DATAGRAM frames
can be packed into a single QUIC packet, the sender SHOULD group them by flow
identifier to promote fate-sharing within a specific flow and improve the
ability to process batches of datagram messages efficiently on the receiver.


# Flow Identifier Allocation {#flow-id-alloc}

Implementations of HTTP/3 that support the DATAGRAM extension MUST provide a
flow identifier allocation service. That service will allow applications
co-located with HTTP/3 to request a unique flow identifier that they can
subsequently use for their own purposes. The HTTP/3 implementation will then
parse the flow identifier of incoming DATAGRAM frames and use it to deliver the
frame to the appropriate application.

Even-numbered flow identifiers are client-initiated, while odd-numbered flow
identifiers are server-initiated. This means that an HTTP/3 client
implementation of the flow identifier allocation service MUST only provide
even-numbered identifiers, while a server implementation MUST only provide
odd-numbered identifiers. Note that, once allocated, any flow identifier can be
used by both client and server - only allocation carries separate namespaces to
avoid requiring synchronization.


# HTTP/3 DATAGRAM Frame Format {#format}

When used with HTTP/3, the Datagram Data field of QUIC DATAGRAM frames uses the
following format (using the notation from the "Notational Conventions" section
of {{QUIC}}):

~~~
HTTP/3 DATAGRAM Frame {
  Flow Identifier (i),
  HTTP/3 Datagram Payload (..),
}
~~~
{: #h3-datagram-format title="HTTP/3 DATAGRAM Frame Format"}

Flow Identifier:

: A variable-length integer indicating the Flow Identifier of the datagram (see
{{flow-id}}).

HTTP/3 Datagram Payload:

: The payload of the datagram, whose semantics are defined by individual
applications. Note that this field can be empty.

Endpoints MUST treat receipt of a DATAGRAM frame whose payload is too short to
parse the flow identifier as a connection error of type PROTOCOL_VIOLATION.


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
SETTINGS parameter.  Doing so allows the client to use HTTP/3 datagrams in
0-RTT packets.  When servers decide to accept 0-RTT data, they MUST send a
H3_DATAGRAM SETTINGS parameter greater or equal to the value they sent to the
client in the connection where they sent them the NewSessionTicket
message.  If a client stores the value of the H3_DATAGRAM SETTINGS parameter
with their 0-RTT state, they MUST validate that the new value of the
H3_DATAGRAM SETTINGS parameter sent by the server in the handshake is greater
or equal to the stored value; if not, the client MUST terminate the connection
with error H3_SETTINGS_ERROR.


# Datagram-Flow-Id Header Field Definition {#header}

"Datagram-Flow-Id" is a Item Structured
Field {{!STRUCT-FIELD=I-D.ietf-httpbis-header-structure}}. Its value MUST be a
List of Integers. Its ABNF is:

~~~
  Datagram-Flow-Id = sf-list
~~~

The "Datagram-Flow-Id" header field is used to associate one or more datagram
flow identifiers with an HTTP message. As a simple example using a single
identifier, the definition of an HTTP method could instruct the client to use
its flow identifier allocation service to allocate a new flow identifier, and
then the client will add the "Datagram-Flow-Id" header field to its request to
communicate that value to the server. In this example, the resulting header
field could look like:

~~~
  Datagram-Flow-Id = 2
~~~

One element in the list is allowed to be unnamed, but all but one elements
MUST carry a name. The name is conveyed using a true boolean parameter
(parameters are defined in Section 3.1.2 of {{STRUCT-FIELD}}) which MUST
be the first parameter of that element. The ordering of the list does not
carry any semantics. For example, an HTTP method that wishes to use four
datagram flow identifiers for the lifetime of its request stream could look
like this:

~~~
  Datagram-Flow-Id = 42, 44; ecn-ect0, 48; ecn-ce, 46; ecn-ect1
~~~

In this example, 42 is the unnamed flow identifier, 44 represents "ecn-ect0",
46 represents "ecn-ect1" and 48 represents "ecn-ce".

Even if a sender attempts to communicate the meaning of a flow identifier
before it uses it in an HTTP/3 datagram, it is possible that its peer will
receive an HTTP/3 datagram with a flow identifier that it does not know as it
has not yet received the corresponding "Datagram-Flow-Id" header field. (For
example, this could happen if the QUIC STREAM frame that contains the
"Datagram-Flow-Id" header field is reordered and arrives afer the DATAGRAM
frame.) Endpoints MUST NOT treat that scenario as an error; they MUST either
silently discard the datagram or buffer it until they receive the
"Datagram-Flow-Id" header field.

Distinct HTTP requests MAY refer to the same flow identifier in their
respective "Datagram-Flow-Id" header fields.

Note that integer structured fields can only encode values up to 10^15-1,
therefore the maximum possible value of the "Datagram-Flow-Id" header field is
lower then the theoretical maximum value of a flow identifier which is 2^62-1
due to the QUIC variable length integer encoding. If the flow identifier
allocation service of an endpoint runs out of values lower than 10^15-1, the
endpoint MUST fail the flow identifier allocation. An HTTP message that
carries a "Datagram-Flow-Id" header field with a flow identifier value above
10^15-1 is malformed.


# HTTP Intermediaries {#intermediaries}

HTTP/3 DATAGRAM flow identifiers are specific to a given HTTP/3 connection.
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
identifier in their respective "Datagram-Flow-Id" header fields, it MUST ensure
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


--- back

# Acknowledgments {#acks}
{:numbered="false"}

The DATAGRAM flow identifier was previously part of the DATAGRAM frame
definition itself, the author would like to acknowledge the authors of
that document and the members of the IETF QUIC working group for their
suggestions. Additionally, the author would like to thank Martin Thomson
for suggesting the use of an HTTP/3 SETTINGS parameter.
