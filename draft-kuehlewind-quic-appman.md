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

This document discusses the applicability of the QUIC transport protocol,
focusing on caveats impacting application protocol development and deployment
over QUIC, and network operations involving QUIC traffic. It also describes
the manageability of QUIC traffic on Internet-connected networks, blah blah
blorg snarf.

--- middle

# Introduction

frontmatter!!

note that this document is only notes. remove this note when we have written a
real draft.

# Terminology

do we need this? mirja says no.

# Applicability of QUIC

note that applicability concerns mainly those aspects of the protocol that
have an impact on the applications that run over QUIC.

## The Necessity of TCP Fallback

UDP blocking about five percent, application must be able to fall back to TCP
over TLS. there are interface issues here: first, must work without stream
multiplexing. second, crypto must work with vanilla TLS 1.3.

## Zero RTT: Here There Be Dragons

Zero RTT data must be treated specially by the application layer: it must be
idempotent, as it may be retransmitted. Note that there is also a replay
attack vector here, so the application layer protocol must also be defined in
such a way to eliminate the utility of a replay attack using zero RTT data.

## Other Shiny Crypto Stuff?

what happens when we have 

## Stream versus Flow Multiplexing

Since stream multiplexing is implemented within the crypto veil, it is not
visible to the network. therefore it is not useful for differentiating streams
in terms of network treatment. in this case, use different five-tuples
instead. this isn't a big deal due to low costs of setting up a new
connection.

# Manageability of QUIC

note that manageability concerns mainly those aspects of the protocol that
have an impact on the operations of devices that forward QUIC packets. here we
concern ourselves primarily with QUIC's wire image, which we define as the
information available in the unencrypted packet header in each QUIC packet,
and the dynamics of that information.

## Versioning

everything can change except the position and meaning of the version field. 

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

## Stateful Treatment of QUIC Traffic

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
