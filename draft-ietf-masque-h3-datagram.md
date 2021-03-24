---
title: Using QUIC Datagrams with HTTP/3
abbrev: HTTP/3 Datagrams
docname: draft-ietf-masque-h3-datagram-latest
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
provide a means to demultiplex application contexts. This document describes how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3. It defines logical flows identified by a non-negative integer that are
present at the start of the DATAGRAM frame payload. Flows are associated with
QUIC streams using the REGISTER_DATAGRAM_FLOW_ID HTTP/3 frame, allowing endpoints to
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
payload. Flows are associated with QUIC streams using the REGISTER_DATAGRAM_FLOW_ID
HTTP/3 frame, allowing endpoints to match unreliable DATAGRAMS frames to the
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
transmitted in order relative to one another. If multiple DATAGRAM frames can be
packed into a single QUIC packet, the sender SHOULD group them by flow to
promote fate-sharing within a specific flow and improve the ability to process
batches of datagram messages efficiently on the receiver.


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

The flow ID allocation service SHOULD also provide a mechanism for applications
to indicate they have completed their usage of a flow ID and will no
longer be using that flow ID, this process is called "retiring" a flow
ID. Applications MUST NOT retire a flow ID until after they
have received confirmation that the peer has also stopped using that flow
ID. The flow ID allocation service MAY reuse previously
retired flow ID once they have ascertained that there are no packets
with DATAGRAM frames using that flow ID still in flight. Reusing flow
IDs can improve performance by transmitting the flow ID using
a shorter variable-length integer encoding.


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


# REGISTER_DATAGRAM_FLOW_ID HTTP/3 Frame Definition {#register-frame}

The REGISTER_DATAGRAM_FLOW_ID frame (type=TBD) is used to register a flow ID
with the QUIC stream that it is sent on. By sending this frame, the sender
indicates that it will process any received datagram with the corresponding
flow ID using semantics specific to this frame. For example, if this is sent on
a client-initiated bidirectional stream that has carried an HTTP request, this
flow ID will now be interpreted as governed by the method of that request.

~~~
REGISTER_DATAGRAM_FLOW_ID Frame {
  Type (i) = TBD,
  Length (i),
  Flow ID (i),
  Encoded Field Section (..),
}
~~~
{: #register-frame-format title="REGISTER_DATAGRAM_FLOW_ID HTTP/3 Frame Format"}

The Type and Length fields follows the definition of HTTP/3 frames from {{H3}}.
The payload consists of:

Flow ID:

: The flow ID to associate with the QUIC stream that carried this frame.

Encoded Field Section:

: QPACK-encoded fields. This allows extensions to optionally attach headers to
their flow IDs.

Note that these registrations are unilateral and unidirectional: the sender of
the frame unilateraly defines the semantics it will apply to the datagrams it
receives. If a mechanism using this feature wants to send datagrams of a given
flow ID in both directions, this frame will need to be exchanged in both
directions.


# RELIABLE_DATAGRAM HTTP/3 Frame Definition {#reliable-datagram-frame}

The RELIABLE_DATAGRAM frame (type=TBD) is used to send datagrams over QUIC
streams when QUIC datagrams are unavailable or undesirable. Datagrams
transmitted over streams using this frame have the same semantics as datgrams
sent over the QUIC DATAGRAM frame.

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


# HTTP/2 Support

We can provide DATAGRAM support in HTTP/2 by defining the
REGISTER_DATAGRAM_FLOW_ID and RELIABLE_DATAGRAM frames in HTTP/2.

TODO: Refactor this document into "HTTP Datagrams" with definitions for
HTTP/2 and HTTP/3.


# HTTP Intermediaries {#intermediaries}

HTTP/3 DATAGRAM flows are specific to a given HTTP/3 connection.
However, in some cases, an HTTP request may travel across multiple HTTP
connections if there are HTTP intermediaries involved; see Section 2.3 of
{{!RFC7230}}.

If an intermediary has sent the H3_DATAGRAM SETTINGS parameter with a value of
1 it MUST NOT blindly forward HTTP/3 frames. The intermediary MUST parse
received REGISTER_DATAGRAM_FLOW_ID and RELIABLE_DATAGRAM frames and act on
them. If the intermediary wishes to forward datagrams from one connection to
another, it MUST generate flow IDs on the outbound connection using its flow ID
allocation service.


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


--- back

# Acknowledgments {#acks}
{:numbered="false"}

The DATAGRAM flow identifier was previously part of the DATAGRAM frame
definition itself, the author would like to acknowledge the authors of
that document and the members of the IETF MASQUE working group for their
suggestions. Additionally, the author would like to thank Martin Thomson
for suggesting the use of an HTTP/3 SETTINGS parameter.
