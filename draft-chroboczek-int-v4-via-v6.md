---
title: "IPv4 routes with an IPv6 next hop"
abbrev: "v4-via-v6"
category: std
submissiontype: IETF

docname: draft-chroboczek-int-v4-via-v6-latest
v: 3
area: INT
workgroup: WG Working Group
keyword: Internet-Draft
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

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

normative:
  rfc7600:

informative:
  rfc4861:
  rfc0826:
  rfc7404:
  rfc0792:
  rfc1191:
  rfc4821:
  IANA-IPV4-REGISTRY:
    title: IP for Smart Objects (IPSO)
    author:
    - org:
    date: false
    seriesinfo:
      Web: https://www.iana.org/assignments/iana-ipv4-special-registry/


--- abstract

This document discusses implications and considerations for routes to an IPv4 prefix with an IPv6 next-hop, to allow IPv4 traffic to flow through interfaces
and devices that have not been assigned an IPv4 address.

--- middle

# Introduction

The role of a routing protocol is to build a routing table, a data
structure that maps network prefixes in a given family (IPv4 or IPv6)
to next hops, pairs of an outgoing interface and a neighbor's
network address, for example:


        destination                      next hop
      2001:db8:0:1::/64               eth0, fe80::1234:5678
      203.0.113.0/24                  eth0, 192.0.2.1

When a packet is routed according to a given routing table entry, the
forwarding plane typically uses a neighbor discovery protocol (the
Neighbor Discovery protocol (ND) [RFC4861] in the case of IPv6, the
Address Resolution Protocol (ARP) [RFC0826] in the case of IPv4) to
map the next-hop address to a link-layer address (a "MAC address"),
which is then used to construct the link-layer frames that
encapsulate forwarded packets.

It is apparent from the description above that there is no
fundamental reason why the destination prefix and the next-hop
address should be in the same address family: there is nothing
preventing an IPv6 packet from being routed through a next hop with
an IPv4 address (in which case the next hop's MAC address will be
obtained using ARP), or, conversely, an IPv4 packet from being routed
through a next hop with an IPv6 address.  (In fact, it is even
possible to store link-layer addresses directly in the next-hop entry
of the routing table, which is commonly done in networks using the
OSI protocol suite).

The case of routing IPv4 packets through an IPv6 next hop is
particularly interesting, since it makes it possible to build
networks that have no IPv4 addresses except at the edges and still
provide IPv4 connectivity to edge hosts.  In addition, since an IPv6
next hop can use a link-local address that is autonomously
configured, the use of such routes enables a mode of operation where
the network core has no statically assigned IP addresses of either
family, which significantly reduces the amount of manual
configuration required.  (See also [RFC7404] for a discussion of the
issues involved with such an approach.)

We call a route towards an IPv4 prefix that uses an IPv6 next hop a
"v4-via-v6" route.

This document discusses the protocol design and operations implications
of such routes, and is designed to be used as a reference for future
documents.

{ Editor note, to be removed before publication. This document is heavily based
on draft-ietf-babel-v4viav6. When draft-ietf-babel-v4viav6 was
going through IESG eval, Warren raised concerns that something this
fundamental deserved to be documented in a separate, standalone document, so
that it can be more fully discussed, and, more importantly, referenced
cleanly in the future. }

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Operational Considerations

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

# Security Considerations

TODO.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
