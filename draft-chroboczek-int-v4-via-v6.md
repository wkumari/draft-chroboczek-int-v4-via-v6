---
title: "IPv4 routes with an IPv6 next hop"
abbrev: "v4-via-v6"
category: std
submissiontype: IETF
ipr: trust200902

docname: draft-chroboczek-intarea-v4-via-v6-latest
v: 3
area: "Internet Engineering Steering Group"
workgroup: "Internet Area"
keyword: Internet-Draft
venue:
  group: "Internet Area"
  type: "Area"
  mail: ""
  arch: ""
  github: "wkumari/draft-chroboczek-int-v4-via-v6"
  latest: "https://wkumari.github.io/draft-chroboczek-int-v4-via-v6/draft-chroboczek-int-v4-via-v6.html"

author:
 -
    fullname: Juliusz Chroboczek
    organization: IRIF, University of Paris
    street:
    - Case 7014
    - 75205 Paris Cedex 13
    - France
    email: jch@irif.fr

 -
    name: Warren Kumari
    ins: W. Kumari
    organization: Google, LLC
    email: warren@kumari.net

 -
    name: Toke Høiland-Jørgensen
    organization: Red Hat
    email: toke@toke.dk

normative:
  RFC7600:

informative:
  RFC0792:
  RFC0826:
  RFC1191:
  RFC4821:
  RFC4861:
  RFC7404:
  RFC8950:
  IANA-IPV4-REGISTRY:
    title: IANA IPv4 Address Registry
    author:
    - org:
    date: false
    seriesinfo:
      Web: https://www.iana.org/assignments/iana-ipv4-special-registry/


--- abstract

We propose "v4-via-v6" routing, a technique that uses IPv6 next-hop
addresses for routing IPv4 packets, thus making it possible to route IPv4
packets across a network where routers have not been assigned IPv4
addresses.  We describe the technique, and discuss its operational
implications.

{ Editor note: This document was originally accidentally published as draft-chroboczek-int-v4-via-v6, and later renamed to draft-chroboczek-intarea-v4-via-v6 (note int vs intarea). The Github repo
uses the old name. }

--- middle

# Introduction

The dominant form of routing in the Internet is next-hop routing, where
a routing protocol constructs a routing table which is used by
a forwarding process to forward packets.  The routing table is a data
structure that maps network prefixes in a given family (IPv4 or IPv6) to
next hops, pairs of an outgoing interface and a neighbor's network
address, for example:


        destination                      next hop
      2001:db8:0:1::/64               eth0, fe80::1234:5678
      203.0.113.0/24                  eth0, 192.0.2.1

When a packet is routed according to a given routing table entry, the
forwarding plane uses a neighbor discovery protocol (the Neighbor
Discovery protocol (ND) [RFC4861] in the case of IPv6, the Address
Resolution Protocol (ARP) [RFC0826] in the case of IPv4) to map the
next-hop address to a link-layer address (a "MAC address"), which is then
used to construct the link-layer frames that encapsulate forwarded
packets.

It is apparent from the description above that there is no fundamental
reason why the destination prefix and the next-hop address should be in
the same address family: there is nothing preventing an IPv6 packet from
being routed through a next hop with an IPv4 address (in which case the
next hop's MAC address will be obtained using ARP), or, conversely, an
IPv4 packet from being routed through a next hop with an IPv6 address.
(In fact, it is even possible to store link-layer addresses directly in
the next-hop entry of the routing table, thus avoiding the use of an
address resolution protocol altogether, which is commonly done in networks
using the OSI protocol suite).

The case of routing IPv4 packets through an IPv6 next hop is
particularly interesting, since it makes it possible to build
networks that have no IPv4 addresses except at the edges and still
provide IPv4 connectivity to edge hosts. In addition, since an IPv6
next hop can use a link-local address that is autonomously
configured, the use of such routes enables a mode of operation where
the network core has no statically assigned IP addresses of either
family, which significantly reduces the amount of manual
configuration required.  (See also [RFC7404] for a discussion of the
issues involved with such an approach.)

We call a route towards an IPv4 prefix that uses an IPv6 next hop a
"v4-via-v6" route.

{{RFC8950}} discusses advertising of IPv4 NLRI with a next-hop address that
belongs to the IPv6 protocol, but confines itself to how this is carried and
advertised in the BGP protocol.

This document is much more general, and discusses the concept of v4-via-v6
routes themselves, the design and operational considerations, and the
implications of using them.

{ Editor note, to be removed before publication. This document is heavily based
on draft-ietf-babel-v4viav6. When draft-ietf-babel-v4viav6 was
going through IESG eval, Warren raised concerns that something this
fundamental deserved to be documented in a separate, standalone document, so
that it can be more fully discussed, and, more importantly, referenced
cleanly in the future.}


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Operation

Next-hop routing is implemented by two separate components, the routing
protocol and the forwarding plane, that communicate through a shared
data structure, the routing table.

## Structure of the routing table

The routing table is a data structure that maps address prefixes to
next-hops, pairs of the form (interface, address).  In traditional
next-hop routing, the routing table maps IPv4 prefixes to IPv4 next hops,
and IPv6 addresses to IPv6 next hops.  With v4-via-v6 routing, the routing
table is extended so that an IPv4 prefix may map to either an IPv4 or an
IPv6 next hop.

## Operation of the forwarding plane

The forwarding plane is the part of the routing implementation that is
executed for every forwarded packet.  As a packet arrives, the forwarding
plane consults the routing table, selects a single route matching the
packet, determines the next-hop address, and forwards the packet to the
next-hop address.

With v4-via-v6 routing, the address family of the next-hop address is no
longer determined by the address family of the prefix: since the routing
table may map an IPv4 prefix to either an IPv4 or an IPv6 next-hop, the
forwarding plane must be able to determine, on a per-packet basis, whether
the next-hop address is an IPv4 or an IPv6 address, and to use that
information in order to choose the right address resolution protocol to
use (ARP for IP4, ND for IPv6).

## Operation of routing protocols

The routing protocol is the part of the routing implementation that is
executed asynchronously from the forwarding plane, and whose role is to
build the routing table.  Since v4-via-v6 routing is a generalisation of
traditional next-hop routing, v4-via-v6 can interoperate with existing
routing protocols: a traditional routing protocol produces a traditional
next-hop routing table, which can be used by an implementation supporting
v4-via-v6 routing.

However, in order to use the additional flexibility provided by v4-via-v6
routing, routing protocols will need to be extended with the ability to
populate the routing table with v4-via-v6 routes when an IPv4 address is
not available or when the available IPv4 addresses are not suitable for
use as a next-hop (e.g., not stable enough).

### Distance-vector routing protocols

### Link-state routing protocols

# ICMP Considerations

The Internet Control Message Protocol (ICMPv4, or simply ICMP)
[RFC0792] is a protocol related to IPv4 that is primarily used to
carry diagnostic and debugging information.  ICMPv4 packets may be
originated by end hosts (e.g., the "destination unreachable, port
unreachable" ICMPv4 packet), but they may also be originated by
intermediate routers (e.g., most other kinds of "destination
unreachable" packets).

Some protocols deployed in the Internet rely on ICMPv4 packets sent
by intermediate routers.  Most notably, path MTU Discovery (PMTUd)
[RFC1191] is an algorithm executed by end hosts to discover the
maximum packet size that a route is able to carry.  While there exist
variants of PMTUd that are purely end-to-end [RFC4821], the variant
most commonly deployed in the Internet has a hard dependency on
ICMPv4 packets originated by intermediate routers: if intermediate
routers are unable to send ICMPv4 packets, PMTUd may lead to
persistent black-holing of IPv4 traffic.

Due to this kind of dependency, every router that is able to
forward IPv4 traffic SHOULD be able originate ICMPv4 traffic.  Since
the extension described in this document enables routers to forward
IPv4 traffic received over an interface that has not been assigned an
IPv4 address, a router implementing this extension MUST be able to
originate ICMPv4 packets even when the outgoing interface has not
been assigned an IPv4 address.

In such a situation, if the router has an interface that has been
assigned an IPv4 address (other than the loopback address), or if an
IPv4 address has been assigned to the router itself (to the "loopback
interface"), then that IPv4 address may be used as the source of
originated ICMPv4 packets.  If no IPv4 address is available, the
router could use the experimental mechanism described in Requirement
R-22 of Section 4.8 [RFC7600], which consists of using the dummy
address 192.0.0.8 as the source address of originated ICMPv4 packets.
Note however that using the same address on multiple routers may
hamper debugging and fault isolation, e.g., when using the
"traceroute" utility.

{Editor note: It would be surprising to many operators to see something like:

~~~~~~~~
> $ traceroute -n 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 64 hops max, 52 byte packets
 1  192.168.0.1  1.894 ms  1.953 ms  1.463 ms
 2  192.0.0.8  9.012 ms  8.852 ms  12.211 ms
 3  192.0.0.8  8.445 ms  9.426 ms  9.781 ms
 4  192.0.0.8  9.984 ms  10.282 ms  10.763 ms
 5  192.0.0.8  13.994 ms  13.031 ms  12.948 ms
 6  192.0.0.8  27.502 ms  26.895 ms
 7  8.8.8.8  26.509 ms
~~~~~~~~

Is this a problem though? If this becomes common practice, will operators
just come to understand that the repeated 192.0.0.8 is not actually a looping
packet, but rather that the packet is (probably!) making forward progress?
What if it goes:
`192.168.0.1 -> 192.0.0.8 -> 10.10.10.10 -> 192.0.0.8 -> 172.16.14.2 -> dest?`

In addition, see RFC5837, and, as a side note, Bill Fenner is working on an update to this document.
}

{ Editor note / question:
192.0.0.8 is assigned in the [IANA-IPV4-REGISTRY] as the "IPv4 dummy address".
It may be used as a Source Address, and was requested in [RFC7600] to be used
as the IPv4 source address when synthesizing an ICMPv4 packet to mirror an
ICMPv6 error message.  This is all fine and good - but, 192.0.0.0/24 is
commonly considered a bogon or martian

Example (from a Juniper router):

~~~~~~~~
wkumari@rtr2.pao> show route martians

inet.0:
             0.0.0.0/0 exact -- allowed
             0.0.0.0/8 orlonger -- disallowed
             127.0.0.0/8 orlonger -- disallowed
             192.0.0.0/24 orlonger -- disallowed
             240.0.0.0/4 orlonger -- disallowed
             224.0.0.0/4 exact -- disallowed
             224.0.0.0/24 exact -- disallowed
~~~~~~~~

This means that these packets are likely to be filtered in many places, and
so ICMP packets with this source address are likely to be dropped. Is this a
major issue? Would requesting another address be a better solution? Would it help? If it were to be allocated from some more global pool, it would still
likely require "magic" to allow it to pass BCP38 filters.
}

# Implementation Status
( This section to be removed before publication. )

As this document does not really define a protocol, this implementation status
section is much less formal. Instead, it is being used as a place to list implementations which are known to support this functionality, examples, notes, etc. This information is provided as a guide to the reader, and is not intended to be a complete list, nor endorsement, etc. If you know of an implementation which is not listed, please let the authors know.

## Mikrotik RouterOS

Mikrotik RouterOS Version 7.11beta2.
{Editor note: I believe it has been supported for a very ling time, but this was the version I tested with. Reminder to self to ask Mikrotik when it was added.}

As an example:
```
[wkumari@Dulles-CCR] /ip/route> print
Flags: D - DYNAMIC; I - INACTIVE, A - ACTIVE; c - CONNECT, s - STATIC, d - DHCP, v - VPN; H - HW-OFFLOADED
Columns: DST-ADDRESS, GATEWAY, DISTANCE
#      DST-ADDRESS       GATEWAY                             DISTANCE
0  As  192.0.2.0/24      fe80::201:5cff:feb2:1646%1_Comcast         1
```

## Arista EOS


Arista has supported static IPv4 routes with IPv6 nexthops since EOS-4.30.1.


# Security Considerations

The techniques described in this document make routing more flexible by
allowing IPv4 routes to propagate across a section of a network that has
only been assigned IPv6 addresses.  This additional flexibility might
invalidate otherwise reasonable assumptions made by network
administrators, which could potentially cause security issues.

For example, if an island of IPv4-only hosts is separated from the IPv4
Internet by routers that have not been assigned IPv4 addresses, a network
administrator might reasonably assume that the IPv4-only hosts are
unreachable from the IPv4 Internet.  This assumption is broken if the
intermediary routers implement v4-via-v6 routing, which might make the
IPv4-only hosts reachable from the IPv4 Internet.  If this is not
desirable, then the network administrator must filter out the undesirable
traffic in the forwarding plane by implementing suitable packet filtering
rules.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
