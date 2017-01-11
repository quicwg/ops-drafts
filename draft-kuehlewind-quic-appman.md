---
title: Applicability and Management of the QUIC Transport Protocol
docname: draft-kuehlewind-quic-appman-latest
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
  I-D.ietf-quic-transport:

informative:
  I-D.trammell-plus-statefulness:

--- abstract

This document discusses the applicability and manageability of the QUIC
transport protocol, focusing on caveats impacting application protocol
development and deployment over QUIC, and network operations involving QUIC
traffic.

--- middle

# Introduction

[EDITOR'S NOTE: This document contains notes toward a -00 revision of the QUIC
applicability and manageability statement. Write some frontmatter, then remove
this note when we have written a real draft.]

# Applicability of QUIC

The QUIC transport is more suited to certain applications and deployment
scenarios than others. The applicability of the protocol concerns mainly those
aspects of the protocol that have an impact on the applications that run over
QUIC. In the following subsections we discuss specific caveats to QUIC's
applicability, and issues that application developers must consider when using
QUIC as a transport for their application.

## The Necessity of TCP Fallback

QUIC uses UDP as a substrate for userspace implementation and port numbers for
NAT and middlebox traversal. While there is no evidence of widespread,
systematic disadvantage of UDP traffic compared to TCP {{Edeline16}},
somewhere between three {{Trammell16-RIPE}} five {{Swett16-MAPRG}} percent of
networks simply block UDP traffic. All applications running on top of QUIC
must therefore either be prepared to accept connectivity failure on such
networks, or be engineered to fall back to TLS, or TLS-equivalent crypto, over
TCP. These applications must operate, perhaps with impaired functionality, in
the absence of features provided by QUIC not present in TLS over TCP: stream
multiplexing,

We hope that the deployment of a proposed standard version of the QUIC
protocol will provide an incentive for these networks to permit QUIC traffic.
Indeed, the ability to treat QUIC traffic statefully as in {{statefulness}}
removes one network management incentive to block this traffic. In the
intermediate-term future, QUIC may therefore be able to evolve into a general-
purpose transport protocol for all applications, without this caveat.

## Zero RTT: Here There Be Dragons

QUIC provides for 0-RTT connection establishment (see section 3.2 of 
{{I-D.ietf-quic-transport}}). However, data in the frames contained in the
first packet of a such a connection must be treated specially by the
application layer. Since a retransmission of these frames resulting from a
lost acknowledgment may cause the data to appear twice, either the
application- layer protocol has to be designed such that all such data is
treated as idempotent, or there must be some application-layer mechanism for
recognizing spuriously retransmitted frame and dropping them.

[EDITOR'S NOTE: discuss defenses against replay attacks using 0-RTT data.]

## Stream versus Flow Multiplexing

QUIC's stream multiplexing feature allows applications to run multiple streams
over a single connection associated at a point in time with a single five-
tuple. Streams are meaningful only to the application; since stream
information is carried inside QUIC's encryption boundary, no information about
the stream(s) whose frames are carried by a given packet is visible to the
network, so stream multiplexing is not useful for differentiating streams in
terms of network treatment.

Different QUIC traffic requiring different network treatment should therefore
be carried over different five-tuples (i.e. multiple connections). If this
functionality is deemed important, the protocol's rebinding functionality (see
section 3.7 of {{I-D.ietf-quic-transport}}) could be extended to allow
multiple five-tuples to share a connection ID simultaneously, instead of
sequentially.

# Manageability of QUIC

The properties of transport-layer protocols have an effect on the operations
and management of the networks that carry them. This section discusses those
aspects of the QUIC transport protocol that have an impact on the design and
operation of devices that forward QUIC packets. This section is concerned
primarily with QUIC's unencrypted wire image, which we define as the
information available in the packet header in each QUIC packet, and the
dynamics of that information.

## Versioning

QUIC is a versioned protocol. Everything about the header format can change
except the mechanism by which a receiver can determine whether and where a
version number is present, and the meaning of the version number field itself.

The rest of this document is concerned with the public header structure of the version of the QUIC transport document that is current as of this writing.

## Version xxx QUIC Public Header Structure

in the current version of the QUIC protocol, here's what is available and when:

- version 
- connection ID (always in version negotiation and public reset, otherwise optional)
- packet number (always except version negotiation and public reset)
- flags (always, but might have different meanings, expect version flag)
- diversification nonce (wtfis????)

## Integrity Protection of the Wire Image

no meddleboxes

## Connection ID and Rebinding

a flow might change one of its IP addresses but keep the same connection ID

## Packet Numbers

note that packet numbers are monotonically increasing, but that packets
containing retransmissions as well as packets containing only acknowledgments
will get new packet numbers. Pure control and retransmission packets are
impossible to distinguish on the wire.

## Stateful Treatment of QUIC Traffic {#statefulness}

borrow from {{I-D.trammell-plus-statefulness}}. connection ID echo as
association/confirmation signal. public reset as partial one-way stop. note
that adding two-way stop as in {{I-D.trammell-plus-statefulness}} has nice
properties.

## Measuring QUIC Traffic

look at the packet numbers, gaps mean upstream loss. out-of-order always means
upstream reordering. unlike with TCP, there is no way to measure RTT
passively.

note that addition of a simple packet number echo would allow passive RTT
measurement and partial passive downstream loss/reordering measurement. packet
number echo can be sampled at the echo-side (i.e. one-in-N packets or 1/p
packets can carry an echo) for efficiency tradeoff, if necessary.

look at the connection IDs for partial tracking of mobility events.

# Open Issues in the QUIC Transport Protocol

note public reset changes for state management may be desirable.

note packet number echo for measurement may be desirable.

note that fiddly-ass packet header structures are asking for hard-to-debug trouble.

# IANA Considerations

This document has no actions for IANA. 

# Security Considerations

write me

# Acknowledgments

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.
