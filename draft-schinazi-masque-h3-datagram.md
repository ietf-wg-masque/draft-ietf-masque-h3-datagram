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


--- abstract

The QUIC DATAGRAM extension provides application protocols running over QUIC
with a mechanism to send unreliable data while leveraging the security and
congestion-control properties of QUIC. However, QUIC DATAGRAM frames do not
provide a means to demultiplex application contexts. This document defines how
to use QUIC DATAGRAM frames when the application protocol running over QUIC is
HTTP/3 by adding an identifier at the start of the frame payload.

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
frame payload.

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
applications.


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

Even flow identifiers are client-initiated, while odd flow identifiers are
server-initiated. This means that an HTTP/3 client implementation of the
flow identifier allocation service MUST only provide even identifiers, while
a server implementation MUST only provide odd identifiers. Note that, once
allocated, any flow identifier can be used by both client and server - only
allocation carries separate namespaces to avoid requiring synchronization.


# The H3_DATAGRAM HTTP/3 SETTINGS Parameter {#setting}

Implementations of HTTP/3 that support this mechanism can indicate that to
their peer by sending the H3_DATAGRAM SETTINGS parameter with a value of 1.
The value of the H3_DATAGRAM SETTINGS parameter MUST be either 0 or 1. A value
of 0 indicates that this mechanism is not supported. An endpoint that receives
the H3_DATAGRAM SETTINGS parameter with a value that is neither 0 or 1 MUST
terminate the connection with error H3_SETTINGS_ERROR.

And endpoint that sends the H3_DATAGRAM SETTINGS parameter with a value of 1
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


# Security Considerations {#security}

This document currently does not have additional security considerations beyond
those defined in {{QUIC}} and {{DGRAM}}.


# IANA Considerations {#iana}

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
that document and the members of the IETF QUIC working group for their
suggestions. Additionally, the author would like to thank Martin Thomson
for suggesting the use of an HTTP/3 SETTINGS parameter.
