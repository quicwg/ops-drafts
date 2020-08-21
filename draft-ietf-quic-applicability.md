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
    org: Ericsson
    email: mirja.kuehlewind@ericsson.com
  -
    ins: B. Trammell
    name: Brian Trammell
    org: Google
    email: ietf@trammell.ch
    street: Gustav-Gull-Platz 1
    city: 8004 Zurich
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
  Hatonen10:
    title: An experimental study of home gateway characteristics (Proc. ACM IMC 2010)
    date: 2010-10
    author:
      -
        ins: S. Hatonen
      -
        ins: A. Nyrhinen
      -
        ins: L. Eggert
      -
        ins: S. Strowes
      -
        ins: P. Sarolahti
      -
        ins: M. Kojo
  I-D.nottingham-httpbis-retry:
--- abstract

This document discusses the applicability of the QUIC transport protocol,
focusing on caveats impacting application protocol development and deployment
over QUIC. Its intended audience is designers of application protocol mappings
to QUIC, and implementors of these application protocols.

--- middle

# Introduction

QUIC {{!QUIC=I-D.ietf-quic-transport}} is a new transport protocol providing a
number of advanced features. While initially designed for the HTTP use case,
like most transports it is intended for use with a much wider variety of
applications. QUIC is encapsulated in UDP. The version of QUIC that is currently
under development will integrate TLS 1.3 {{!TLS13=I-D.ietf-quic-tls}} to encrypt
all payload data and most control information. HTTP operating over QUIC is known
as HTTP/3.

This document provides guidance for application developers that want to use
the QUIC protocol without implementing it on their own. This includes general
guidance for applications operating over HTTP/3 or directly over QUIC. For
specific guidance on how to integrate HTTP/3 with QUIC, see
{{?QUIC-HTTP=I-D.ietf-quic-http}}.

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
implemented in the application layer if needed.

Further, TCP implementations and network paths often do not support the Fast
Open option, which is analogous to 0-RTT session resumption. Even if Fast Open
successfully operates end-to-end, it is limited to a single packet of payload,
unlike QUIC 0-RTT.

Note that there is some evidence of middleboxes blocking SYN data even if TFO
was successfully negotiated (see {{PaaschNanog}}).

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
Applications MUST be designed, and their data MUST be framed, such that multiple
reception of idempotent data is recognized as such by the receiver. Applications
that cannot treat data that may appear in a 0-RTT connection establishment as
idempotent MUST NOT use 0-RTT establishment. For these reason the QUIC transport
SHOULD provide some or all of the following interfaces to applications:

* indicate if 0-RTT support is in general desired, which implies that lack of
PFS is acceptable for some data;

* an indication when 0RTT data for both egress and ingress, so that both sender
and receiver understand the properties of the communication channel when the
data is sent; and/or

* whether rejected 0-RTT data should be retransmitted or withdrawn.

Some TLS implementations may offer replay protection, which may mitigate some
of these issues.

## Session resumption versus Keep-alive {#resumption-v-keepalive}

Because QUIC is encapsulated in UDP, applications using QUIC must deal with
short idle timeouts. Deployed stateful middleboxes will generally establish
state for UDP flows on the first packet state, and keep state for much shorter
idle periods than for TCP. According to a 2010 study ({{Hatonen10}}), UDP
applications can assume that any NAT binding or other state entry will be
expired after just thirty seconds of inactivity.

By using a Connection ID, QUIC is designed to be robust to NAT address
rebinding after a timeout. However, some QUIC connections may not be robust to
rebinding because the routing infrastructure (in particular, load balancers)
uses the address/port four-tuple to direct traffic. Furthermore, middleboxes
with functions other than address translation may still affect the path. In
particular, firewalls will often not admit server traffic for which it has not
kept state for corresponding packets from the client.

A QUIC application has three strategies to deal with this issue:

- Ignore it, if the application-layer protocol consists only of interactions
  with no or very short idle periods, or the protocol's resistance to NAT
  rebinding is sufficient.
- Ensure there are no long idle periods.
- Resume the session after a long idle period, using 0-RTT resumption when
  appropriate.

The first strategy is the easiest, but it only applies to certain applications.

Either the server or the client in a QUIC application can send PING frames as
keep-alives, to prevent the connection and any on-path state from timing out.
Recommendations for the use of keep-alives are application specific, mainly
depending on the latency requirements and message frequency of the application.
In this case, the application mapping must specify whether the client or server
is responsible for keeping the application alive. Note that sending PING frames
more frequently than every 30 seconds over long idle periods may result in a too
much unproductive traffic and power usage for some situations.

Alternatively, the client (but not the server) can use session resumption
instead of sending keepalive traffic. In this case, a client that wants to send
data to a server over a connection idle longer than the server's idle timeout
(available from the idle_timeout transport parameter) can simply reconnect. When
possible, this reconnection can use 0-RTT session resumption, reducing the
latency involved with restarting the connection. This of course only applies in
cases in which 0-RTT data is safe, when the client is the restarting peer, and
when the data to be sent is idempotent.

The tradeoffs between resumption and keepalive need to be evaluated on a
per-application basis. However, in general applications should use keepalives
only in circumstances where continued communication is highly likely;
{{QUIC-HTTP}}, for instance, recommends using PING frames for keepalive only
when a request is outstanding.

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

Mapping of application data to streams is application-specific and described for
HTTP/s in {{QUIC-HTTP}}. In general data that can be processed independently,
and therefore would suffer from head of line blocking if forced to be received
in order, should be transmitted over different streams. If the application
requires certain data to be received in order, the same stream should
be used for that data. If there is a logical grouping of data chunks or
messages, streams can be reused, or a new stream can be opened for each
chunk/message. If one message is mapped to a single stream, resetting the stream
to expire an unacknowledged message can be used to emulate partial reliability
on a message basis. If a QUIC receiver has maximum allowed concurrent streams
open and the sender on the other end indicates that more streams are needed, it
doesn't automatically lead to an increase of the maximum number of streams by
the receiver. Therefore it can be valuable to expose maximum number of allowed,
currently open and currently used streams to the application to make the mapping
of data to streams dependent on this information.

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


## Flow Control Deadlocks {#flow-control-deadlocks}

Flow control provides a means of managing access to the limited buffers
endpoints have for incoming data.  This mechanism limits the amount of data that
can be in buffers in endpoints or in transit on the network.  However, there are
several ways in which limits can produce conditions that can cause a connection
to either perform suboptimally or deadlock.

Deadlocks in flow control are possible for any protocol that uses QUIC, though
whether they become a problem depends on how implementations consume data and
provide flow control credit.  Understanding what causes deadlocking might help
implementations avoid deadlocks.

Large messages can produce deadlocking if the recipient does not process the
message incrementally.  If the message is larger than flow control credit
available and the recipient does not release additional flow control credit
until the entire message is received and delivered, a deadlock can occur.  This
is possible even where stream flow control limits are not reached because
connection flow control limits can be consumed by other streams.

A common flow control implementation technique is for a receiver to extend
credit to the sender as a the data consumer reads data. In this setting, a
length-prefixed message format makes it easier for the data consumer to leave
data unread in the receiver's buffers and thereby withhold flow control credit.
If flow control limits prevent the remainder of a message from being sent, a
deadlock will result.  A length prefix might also enable the detection of this
sort of deadlock.  Where protocols have messages that might be processed as a
single unit, reserving flow control credit for the entire message atomically
ensures that this style of deadlock is less likely.

A data consumer can read all data as it becomes available to cause the receiver
to extend flow control credit to the sender and reduce the chances of a
deadlock.  However, releasing flow control credit might mean that the data
consumer might need other means for holding a peer accountable for the state it
keeps for partially processed messages.

Deadlocking can also occur if data on different streams is interdependent.
Suppose that data on one stream arrives before the data on a second stream on
which it depends.  A deadlock can occur if the first stream is left unread,
preventing the receiver from extending flow control credit for the second
stream.  To reduce the likelihood of deadlock for interdependent data, the
sender should ensure that dependent data is not sent until the data
it depends on has been accounted for in both stream- and connection- level flow
control credit.

Some deadlocking scenarios might be resolved by cancelling affected streams with
STOP_SENDING or RST_STREAM.  Cancelling some streams results in the connection
being terminated in some protocols.

# Packetization and Latency

QUIC provides an interface that provides multiple streams to the application;
however, the application usually cannot control how data transmitted over one
stream is mapped into frames or how those frames are bundled into packets.
By default, QUIC will try to maximally pack packets with one or more stream
data frames to minimize bandwidth consumption and computational costs (see
section 13 of {{!QUIC}}). If there is not enough data available to fill a
packet,
QUIC may even wait for a short time, to optimize bandwidth efficiency instead of
latency. This delay can either be pre-configured or dynamically adjusted based
on the observed sending pattern of the application. If the application requires
low latency, with only small chunks of data to send, it may be valuable to
indicate to QUIC that all data should be send out immediately. Alternatively,
if the application expects to use a specific sending pattern, it can also
provide a suggested delay to QUIC for how long to wait before bundle frames
into a packet.

Similarly, an appliaction has usually no control about the length of a QUIC
packet on the wire. However, QUIC provides the ability to add a padding frame to
impact the packet size. This is mainly used by QUIC itself in the first packet
in order to ensure that the path is capable of transferring packets of at least
a certain size. Additionally, a QUIC implementation can expose an application
layer interface to specify a certain packet size. This can either be used by the
application to force certian packet sizes in specific use cases/networks, or
ensure that all packets are equally sized to conceal potential leakage of
application layer information when the data sent by the application are not
greedy. Note the initial packet must have a minimum size of 1200 bytes
according to the QUIC specification. A receiver of a smaller initial packet may
reject this packet in order to avoid amplification attacks.


# Port Selection and Application Endpoint Discovery {#ports}

In general, port numbers serves two purposes: "first, they provide a
demultiplexing identifier to differentiate transport sessions between the same
pair of endpoints, and second, they may also identify the application protocol
and associated service to which processes connect" {{!RFC6335}}. Note that the
assumption that an application can be identified in the network based on the
port number is less true today, due to encapsulation, mechanisms for dynamic
port assignments as well as NATs.

As QUIC is a general purpose transport protocol, there are no requirements that
servers use a particular UDP port for QUIC in general. For applications with a
fallback to TCP which do not already have an alternate mapping to UDP, the
registration (if necessary) and use of the UDP port number corresponding to
the TCP port already registered for the application is RECOMMENDED. For example,
the default port for HTTP/3 {{QUIC-HTTP}} is UDP port 443, analogous to HTTP/1.1
or HTTP/2 over TLS over TCP.

Applications SHOULD define an alternate endpoint discovery mechanism to allow
the usage of ports other than the default. For example, HTTP/3 ({{QUIC-HTTP}}
sections 3.2 and 3.3) specifies the use of ALPN {{?RFC7301}} for service
discovery which allows the server to use and announce a different
port number. Note that HTTP/3's ALPN token ("h3") identifies not only the
version of the application protocol, but also the binding to QUIC as well
as the version of QUIC itself; this approach allows unambiguous agreement
between the endpoints on the protocol stack in use.

Note that given the prevalence of the assumption in network management
practice that a port number maps unambiguously to an application, the
use of ports that cannot easily be mapped to a registered service name
may lead to blocking or other interference by network elements such as
firewalls that rely on the port number for application identification.


# Connection Migration

QUIC supports connection migration. Even if lower-layer addresses (usually the
4-tuple of IP addresses and ports) changes, QUIC packets can still be associated
with an existing connection based on the Connection ID (see also section
{{connid}}) in the QUIC header, if present. This supports cases where address
information changes due to e.g. NAT rebinding or change of the local interface.
Currently QUIC only supports failover cases. Only one "path" can be used at a
time, and as soon as the new path is validated all traffic will be switched over
to the next path. Of course if an endpoint decided to not use the Connection ID
in short packets (Zero-length Conn ID) for a certain connection, migration is
not supported for that direction of the connection.


# Connection closure

QUIC connections are closed either by expiration of an idle timeout or by an
explicit indication of the application that a connection should be closed
(immediate close). While data could still be received after the immediate close
has been initiated by one endpoint (for a limited time period), the expectation
is that an immediate close was negotiated at the application layer and
therefore no additional data is expected from both sides.

An immidate close will emit an CONNECTION_CLOSE frame. This frames has two sets
of types: one for QUIC internal problems that might lead to connection closure,
and one for closures initiated by the application. An application using QUIC can
define application-specific error codes, e.g. see {{QUIC-HTTP}} section 8.1. In
the case of a grateful shut-down initiated by the application after application
layer negotiation, a NO_ERROR code is expected. Further, the CONNECTION_CLOSE
frame provides an optional reason field, that can be used to append
human-readable information to an error code. Note that QUIC RESET_STREAM and
STOP_SENDING frames provide similar capablities. Usually application error codes
are defined to be applicabile to all three frames.

Alternatively, a QUIC connection can be silently closed by each endpoint
separately after an idle timeout. If enabled as indicated by a transport
parameter in the handshake, the idle timeout is announced for each
endpoint during connection establishment and the effective value for this
connection is the minimum of the two advertised values. An application
therefore should be able to configure its own maximum value as well as
have access to the computed minimum value for this connection. An application
may adjust the maximum idle timeout based on the number of open or expected
connections as shorter timeout values may free-up memory more quickly.
If an application desires to keep the connection open for longer
than the announced timeout, it can send keep-alives messages, or a QUIC
implementation may provide an option to defer the time-out to avoid unnecessary
load, as specified in Section 10.2.2 of {{QUIC}}.
See {{resumption-v-keepalive}} for further guidance on keep-alives.


# Information exposure and the Connection ID {#connid}

QUIC exposes some information to the network in the unencrypted part of the
header, either before the encryption context is established, because the
information is intended to be used by the network. QUIC has a long header that
is used during connection establishment and for other control processes, and a
short header that may be used for data transmission in an established
connection. While the long header always exposes some information (such as the
version and Connection IDs), the short header exposes at most only a single
Connection ID.

Note that the Connection ID in the short header may be omitted. This is a
per-connection configuration option; if the Connection ID is not present, then
the peer omitting the connection ID will use the same local address for the
lifetime of the connection.

## Server-Generated Connection ID

QUIC supports a server-generated Connection ID, transmitted to the client during
connection establishment (see Section 6.1 of {{!QUIC}}). Servers behind load
balancers may need to change the Connection ID during the handshake, encoding
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
most rigorously achieved with encryption. Even when encrypted, a scheme could
embed the unencrypted length of the Connection ID in the Connection ID itself,
instead of remembering it, e.g. by using the first few bits to indicate a
certain size of a well-known set of possible sizes with multiple values that
indicate the same size but are selected randomly.

{{?QUIC_LB=I-D.ietf-quic-load-balancers}} further specified possible algorithms
to generate Connection IDs at load balancers.

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
Alternatively the load balancer can directly offer a Retry services as further
described in {{?QUIC-LB=I-D.ietf-quic-load-balancers}}.

{{?RFC5077}} Section 4 describes an example approach for constructing
TLS resumption tickets that can be also applied for validation tokens,
however, the use of more modern cryptographic algorithms is highly recommended.

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

# Enabling New Versions

QUIC provides integrity protection for its version negotiation process.  This
process assumes that the set of versions that a server supports is fixed.  This
complicates the process for deploying new QUIC versions or disabling old
versions when servers operate in clusters.

A server that rolls out a new version of QUIC can do so in three stages.  Each
stage is completed across all server instances before moving to the next stage.

In the first stage of deployment, all server instances start accepting new
connections with the new version.  The new version can be enabled progressively
across a deployment, which allows for selective testing.  This is especially
useful when the new version is compatible with an old version, because the new
version is more likely to be used.

While enabling the new version, servers do not advertise the new version in any
Version Negotiation packets they send.  This prevents clients that receive a
Version Negotiation packet from attempting to connect to server instances that
might not have the new version enabled.

During the initial deployment, some clients will have received Version
Negotiation packets that indicate that the server does not support the new
version.  Other clients might have successfully connected with the new version
and so will believe that the server supports the new version.  Therefore,
servers need to allow for this ambiguity when validating the negotiated version.

The second stage of deployment commences once all server instances are able
accept new connections with the new version.  At this point, all servers can
start sending the new version in Version Negotiation packets.

During the second stage, the server still allows for the possibility that some
clients believe the new version to be available and some do not.  This state
will persist only for as long as any Version Negotiation packets take to be
transmitted and responded to.  So the third stage can follow after a relatively
short delay.

The third stage completes the process by enabling validation of the negotiation
version as though the new version were disabled.

The process for disabling an old version or rolling back the introduction of a
new version uses the same process in reverse.  Servers disable validation of the
old version, stop sending the old version in Version Negotiation packets, then
the old version is no longer accepted.


# IANA Considerations

This document has no actions for IANA; however, note that {{ports}}
recommends that application bindings to QUIC for applications using
TCP register UDP ports analogous to their existing TCP registrations.

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
