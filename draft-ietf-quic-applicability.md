---
title: Applicability of the QUIC Transport Protocol
abbrev: QUIC Applicability
docname: draft-ietf-quic-applicability-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: B. Trammell
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland

normative:
  RFC2119:

informative:
  Trammell16:
    title: Internet Path Transparency Measurements using RIPE Atlas (RIPE72 MAT presentation)
    author:
      -
        ins: B. Trammell
      -
        ins: M. Kuehlewind
    target: https://ripe72.ripe.net/wp-content/uploads/presentations/86-atlas-udpdiff.pdf
    date: 2016-05-25
  Edeline16:
    title: Using UDP for Internet Transport Evolution (arXiv preprint 1612.07816)
    author:
      -
        ins: K. Edeline
      -
        ins: M. Kuehlewind
      -
        ins: B. Trammell
      -
        ins: E. Aben
      -
        ins: B. Donnet
    target: https://arxiv.org/abs/1612.07816
    date: 2016-12-22
  Swett16:
    title: QUIC Deployment Experience at Google (IETF96 QUIC BoF presentation)
    author:
      -
        ins: I. Swett
    target: https://www.ietf.org/proceedings/96/slides/slides-96-quic-3.pdf
    date: 2016-07-20
  PaaschNanog:
    title: Network Support for TCP Fast Open (NANOG 67 presentation)
    author:
      -
        ins: C. Paasch
    target: https://www.nanog.org/sites/default/files/Paasch_Network_Support.pdf
    date: 2016-06-13
  I-D.nottingham-httpbis-retry:
--- abstract

This document discusses the applicability of the QUIC transport protocol,
focusing on caveats impacting application protocol development and deployment
over QUIC. Its intended audience is designers of application protocol mappings
to QUIC, and implementors of these application protocols.

--- middle

# Introduction

QUIC {{!QUIC=I-D.ietf-quic-transport}} is a new transport protocol currently
under development in the IETF quic working group, focusing on support of
semantics as needed for HTTP/2 {{?QUIC-HTTP=I-D.ietf-quic-http}} such as
stream-multiplexing to avoid head-of-line blocking. Based on current
deployment practices, QUIC is encapsulated in UDP. The version of QUIC that is
currently under development will integrate TLS 1.3
{{!TLS13=I-D.ietf-quic-tls}} to encrypt all payload data and most control
information.

This document provides guidance for application developers that want to use
the QUIC protocol without implementing it on their own. This includes general
guidance for application use of HTTP/2 over QUIC as well as the use of other
application layer protocols over QUIC. For specific guidance on how to
integrate HTTP/2 with QUIC, see {{QUIC-HTTP}}.

In the following sections we discuss specific caveats to QUIC's applicability,
and issues that application developers must consider when using QUIC as a
transport for their application.

## Notational Conventions

The words "MUST", "MUST NOT", "SHOULD", and "MAY" are used in this document.
It's not shouting; when these words are capitalized, they have a special
meaning as defined in {{RFC2119}}.

# The Necessity of Fallback {#fallback}

QUIC uses UDP as a substrate for userspace implementation and port numbers for
NAT and middlebox traversal. While there is no evidence of widespread,
systematic disadvantage of UDP traffic compared to TCP in the Internet
{{Edeline16}}, somewhere between three {{Trammell16}} and five {{Swett16}}
percent of networks simply block UDP traffic. All applications running on top
of QUIC must therefore either be prepared to accept connectivity failure on
such networks, or be engineered to fall back to some other transport protocol.
This fallback SHOULD provide TLS 1.3 or equivalent cryptographic protection,
if available, in order to keep fallback from being exploited as a downgrade
attack. In the case of HTTP, this fallback is TLS 1.3 over TCP.

These applications must operate, perhaps with impaired functionality, in the
absence of features provided by QUIC not present in the fallback protocol. For
fallback to TLS over TCP, the most obvious difference is that TCP does not
provide stream multiplexing and therefore stream multiplexing would need to be
implemented in the application layer if needed. Further, TCP without the TCP
Fast Open extension does not support 0-RTT session resumption. TCP Fast Open
can be requested by the connection initiator but might no be supported by the
far end or could be blocked on the network path. Note that there is some
evidence of middleboxes blocking SYN data even if TFO was successfully
negotiated (see {{PaaschNanog}}).

Any fallback mechanism is likely to impose a degradation of performance;
however, fallback MUST not silently violate the application's expectation of
confidentiality or integrity of its payload data.

Moreover, while encryption (in this case TLS) is inseparably integrated with
QUIC, TLS negotiation over TCP can be blocked. In case it is RECOMMENDED to
abort the connection, allowing the application to present a suitable prompt to
the user that secure communication is unavailable.

# Zero RTT {#zero-rtt}

QUIC provides for 0-RTT connection establishment. This presents opportunities
and challenges for applications using QUIC.

## Thinking in Zero RTT

A transport protocol that provides 0-RTT connection establishment to recently
contacted servers is qualitatively different than one that does not from the
point of view of the application using it. Relative trade-offs between the cost
of closing and reopening a connection and trying to keep it open are
different; see {{resumption-v-keepalive}}.

Applications must be slightly rethought in order to make best use of 0-RTT
resumption. Most importantly, application operations must be divided into
idempotent and non-idempotent operations, as only idempotent operations may
appear in 0-RTT packets. This implies that the interface between the
application and transport layer exposes idempotence either explicitly or
implicitly.

## Here There Be Dragons

Retransmission or (malicious) replay of data contained in 0-RTT resumption
packets could cause the server side to receive two copies of the same data.
This is further described in {{?HTTP-RETRY=I-D.nottingham-httpbis-retry}}.
Data sent during 0-RTT resumption also cannot benefit from perfect forward
secrecy (PFS).

Data in the first flight sent by the client in a connection established with
0-RTT MUST be idempotent (as specified in section 2.1 in {{!QUIC-TLS}}).
Applications MUST be designed, and their data MUST be framed, such that
multiple reception of idempotent data is recognized as such by the
receiverApplications that cannot treat data that may appear in a 0-RTT
connection establishment as idempotent MUST NOT use 0-RTT establishment. For
this reason the QUIC transport SHOULD provide an interface for the application
to indicate if 0-RTT support is in general desired or a way to indicate
whether data is idempotent, and/or whether PFS is a hard requirement for the
application.

## Session resumption versus Keep-alive {#resumption-v-keepalive}

\[EDITOR'S NOTE: see https://github.com/quicwg/ops-drafts/issues/6]

# Use of Streams

QUIC's stream multiplexing feature allows applications to run multiple streams
over a single connection, without head-of-line blocking between streams,
associated at a point in time with a single five-tuple. Stream data is carried
within Frames, where one (UDP) packet on the wire can carry one of multiple
stream frames.

Stream can be independently open and closed, gracefully or by error. If a
critical stream for the application is closed, the application can generate
respective error messages on the application layer to inform the other end or
the higher layer and eventually indicate QUIC to reset the connection. QUIC,
however, does not need to know which streams are critical, and does not
provide an interface to exceptional handling of any stream. There are special
streams in QUIC that are used for control on the QUIC connection, however,
these streams are not exposed to the application.

Mapping of application data to streams is application-specific and described
for HTTP/s in {{QUIC-HTTP}}. In general data that can be processed
independently, and therefore would suffer from head of line blocking, if
forced to be received in order, should be transmitted over different streams.
If there is a logical grouping of those data chunks or messages, stream can be
reused, or a new stream can be opened for each chunk/message. If a QUIC
receiver has maximum allowed concurrent streams open and the sender on the
other end indicates that more streams are needed, it doesn't automatically
lead to an increase of the maximum number of streams by the receiver.
Therefore it can be valuable to expose maximum number of allowed, currently
open and currently used streams to the application to make the mapping of
data to streams dependent on this information.

Further, streams have a maximum number of bytes that can be sent on one
stream. This number is high enough (2^64) that this will usually not be
reached with current applications. Applications that send chunks of data over
a very long period of time (such as days, months, or years), should rather
utilize the 0-RTT session resumption ability provided by QUIC, than trying to
maintain one connection open.

## Stream versus Flow Multiplexing

Streams are meaningful only to the application; since stream information is
carried inside QUIC's encryption boundary, no information about the stream(s)
whose frames are carried by a given packet is visible to the network.
Therefore stream multiplexing is not intended to be used for differentiating
streams in terms of network treatment. Application traffic requiring different
network treatment SHOULD therefore be carried over different five-tuples (i.e.
multiple QUIC connections). Given QUIC's ability to send application data in
the first RTT of a connection (if a previous connection to the same host has
been successfully established to provide the respective credentials), the cost
of establishing another connection is extremely low.

## Packetization and latency

QUIC provides an interface that provides multiple streams to the application;
however, the application usually cannot control how data transmitted over one
stream is mapped into frames or how those frames are bundled into packets.
By default, QUIC will try to maximally pack packets with one or more stream
data frames to minimize bandwidth consumption and computational costs (see
section 8 of {{!QUIC}}). If there is not enough data available to fill a packet,
QUIC may even wait for a short time, to optimize bandwidth efficiency instead of
latency. This delay can either be pre-configured or dynamically adjusted based
on the observed sending pattern of the application. If the application requires
low latency, with only small chunks of data to send, it may be valuable to
indicate to QUIC that all data should be send out immediately. Alternatively,
if the application expects to use a specific sending pattern, it can also
provide a suggested delay to QUIC for how long to wait before bundle frames
into a packet.

## Prioritization

Stream prioritization is not exposed to either the network or the receiver.
Prioritization is managed by the sender, and the QUIC transport should
provide an interface for applications to prioritize streams {{!QUIC}}. Further
applications can implement their own prioritization scheme on top of QUIC: an
application protocol that runs on top of QUIC can define explicit messages
for signaling priority, such as those defined for HTTP/2; it can define rules
that allow an endpoint to determine priority based on context; or it can
provide a higher level interface and leave the determination to the
application on top.

Priority handling of retransmissions can be implemented by the sender in the
transport layer. {{QUIC}} recommends to retransmit lost data before new data,
unless indicated differently by the application. Currently, QUIC only provides
fully reliable stream transmission, which means that prioritization of
retransmissions will be beneficial in most cases, by filling in gaps and freeing
up the flow control window. For partially reliable or unreliable streams,
priority scheduling of retransmissions over data of higher-priority streams
might not be desirable. For such streams, QUIC could either provide an
explicit interface to control prioritization, or derive the prioritization
decision from the reliability level of the stream.

# Graceful connection closure

\[EDITOR'S NOTE: give some guidance here about the steps an application should
take; however this is still work in progress]

# Information exposure and the Connection ID {#connid}

QUIC exposes some information to the network in the unencrypted part of the
header, either before the encryption context is established, because the
information is intended to be used by the network. QUIC has a long header that
is used during connection establishment and for other control processes, and a
short header that may be used for data transmission in an established
connection. While the long header always exposes some information (such as the
version and Connection IDs), the short header exposes at most only a single
Connection ID.

## Server-Generated Connection ID

QUIC supports a server-generated Connection ID, transmitted to the client during
connection establishment (see Section 6.1 of {{!QUIC}}). Servers behind load
balancers may need to  propose a Connection ID during the handshake, encoding
the identity of the server or information about its load balancing pool, in
order to support stateless load balancing. Once the server generates a
Connection ID that encodes its identity, every CDN load balancer would be able
to forward the packets to that server without retaining connection state.

Server-generated connection IDs should seek to obscure any encoding, of routing
identities or any other information. Exposing the server mapping would allow
linkage of multiple IP addresses to the same host if the server also supports
migration. Furthermore, this opens an attack vector on specific servers or
pools.

The best way to obscure an encoding is to appear random to observers, which is
most rigorously achieved with encryption.

## Mitigating Timing Linkability with Connection ID Migration

While sufficiently robust connection ID generation schemes will mitigate
linkability issues, they do not provide full protection. Analysis of
the lifetimes of six-tuples (source and destination addresses as well as the
migrated CID) may expose these links anyway.

In the limit where connection migration in a server pool is rare, it is trivial
for an observer to associate two connection IDs. Conversely, in the opposite
limit where every server handles multiple simultaneous migrations, even an
exposed server mapping may be insufficient information.

The most efficient mitigation for these attacks is operational, either by using
a load balancing architecture that loads more flows onto a single server-side
address, by coordinating the timing of migrations to attempt to increase the
number of simultaneous migrations at a given time, or through other means.

## Using Server Retry for Redirection

QUIC provides a Server Retry packet that can be sent by a server in response to
the Client Initial packet. The server may choose a new Connection ID in that
packet and the client will retry by sending another Client Initial packet with
the server-selected Connection ID. This mechanism can be used to redirect a
connection to a different server, e.g. due to performance reasons or when
servers in a server pool are upgraded gradually, and therefore may support
different versions of QUIC. In this case, it is assumed that all servers
belonging to a certain pool are served in cooperation with load balancers that
forward the traffic based on the Connection ID. A server can choose the
Connection ID in the Server Retry packet such that the load balancer will
redirect the next Client Initial packet to a different server in that pool.

# Use of Versions and Cryptographic Handshake

Versioning in QUIC may change the protocol's behavior completely, except
for the meaning of a few header fields that have been declared to be invariant
{{!QUIC-INVARIANTS=I-D.ietf-quic-invariants}}. A version of QUIC
with a higher version number will not necessarily provide a better service,
but might simply provide a different feature set. As such, an application needs
to be able to select which versions of QUIC it wants to use.

A new version could use an encryption scheme other than TLS 1.3 or higher.
{{QUIC}} specifies requirements for the cryptographic handshake as currently
realized by TLS 1.3 and described in a separate specification
{{!QUIC-TLS=I-D.ietf-quic-tls}}. This split is performed to enable
light-weight versioning with different cryptographic handshakes.

# IANA Considerations

This document has no actions for IANA.

# Security Considerations

See the security considerations in {{!QUIC}} and {{!QUIC-TLS}}; the security
considerations for the underlying transport protocol are relevant for
applications using QUIC, as well.

Application developers should note that any fallback they use when QUIC cannot
be used due to network blocking of UDP SHOULD guarantee the same security
properties as QUIC; if this is not possible, the connection SHOULD fail to
allow the application to explicitly handle fallback to a less-secure
alternative. See {{fallback}}.

# Contributors

Igor Lubashev contributed text to {{connid}} on server-selected Connection IDs.

# Acknowledgments

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research, and
Innovation under contract no. 15.0268. This support does not imply endorsement.
