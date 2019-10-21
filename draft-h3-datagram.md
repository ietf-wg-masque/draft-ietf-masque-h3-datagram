---
title: Using QUIC Datagrams with HTTP/3
abbrev: HTTP/3 Datagrams
docname: draft-schinazi-quic-h3-datagram
category: exp

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

The QUIC DATAGRAM extension {{!I-D.pauly-quic-datagram}} provides application
protocols running over QUIC with a way to send unreliable data while leveraging
the security and congestion-control properties of QUIC. However, QUIC DATAGRAM
frames do not provide a means to demultiplex application contexts using them.
This document defines how to use QUIC DATAGRAM frames when the application
protocol running over QUIC is HTTP/3 {{!I-D.ietf-quic-http}}, by adding an
identifier as the start of the frame payload.


--- middle

# Introduction

The QUIC DATAGRAM extension {{!I-D.pauly-quic-datagram}} provides application
protocols running over QUIC with a way to send unreliable data while leveraging
the security and congestion-control properties of QUIC. However, QUIC DATAGRAM
frames do not provide a means to demultiplex application contexts using them.
This document defines how to use QUIC DATAGRAM frames when the application
protocol running over QUIC is HTTP/3 {{!I-D.ietf-quic-http}}, by adding an
identifier as the start of the frame payload. This design mimics the use of
Stream Types in HTTP/3, which provide a demultiplexing identifier at the start
of each unidirectional stream.


## Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# HTTP/3 DATAGRAM Frame Format

When used with HTTP/3, the Datagram Data field of QUIC DATAGRAM frames uses the
following format:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Flow Identifier (i)                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 HTTP/3 Datagram Payload (*)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #h3-datagram-format title="HTTP/3 DATAGRAM Frame Format"}


# Flow Identifier Allocation

Implementations of HTTP/3 that support the DATAGRAM extension will provide a
flow identifier allocation service. That service will allow application
co-located with HTTP/3 to request a unique Flow Identifier that they can
subsequently use for their own purposes. The HTTP/3 implementation will then
parse the Flow Identifier of incoming DATAGRAM frames and use it to deliver the
frame to the appropriate application.


# Security Considerations

This document currently does not have additional security considerations on top
of the ones defined in {{!I-D.ietf-quic-transport}} and
{{!I-D.pauly-quic-datagram}}.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The DATAGRAM frame identifier was previously part of the DATAGRAM frame
definition itself, the author would like to acknowledge the authors of
that document and the members of the IETF QUIC working group for their
suggestions.
