---
title: "MoQ Relay Hops Extension"
abbrev: "moq-relay-hops"
category: info

docname: draft-lcurley-moq-relay-hops-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
v: 3
area: wit
workgroup: moq

author:
 -
    fullname: Luke Curley
    email: kixelated@gmail.com

normative:
  moqt: I-D.ietf-moq-transport

informative:

--- abstract

This document defines a Relay Hops extension for MoQ Transport {{moqt}}.
Each namespace advertisement carries an ordered list of Hop IDs identifying the relays it has traversed, and a namespace subscription MAY carry a Hop ID to exclude.
Together these allow a relay cluster to detect and avoid routing loops, and allow a downstream subscriber to break ties between multiple paths to the same namespace by preferring the shortest one.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Introduction
{{moqt}} is designed to deliver content end-to-end through a mesh of relays.
A namespace advertisement (PUBLISH_NAMESPACE, {{moqt}} Section 10.15) originates at a publisher and propagates downstream through one or more relays toward interested subscribers, which express interest with SUBSCRIBE_NAMESPACE ({{moqt}} Section 10.18).

In a redundant deployment, relays are interconnected so that the same namespace can reach a given relay over more than one path.
This redundancy is desirable for failover, but it introduces two problems that {{moqt}} does not address:

- **Routing loops**: relay A advertises a namespace to relay B, which advertises it back to A (directly or through a cycle). Without a way to recognize an advertisement it has already seen, a relay will re-advertise it indefinitely.
- **Path selection**: when the same namespace arrives over multiple paths, a relay or subscriber has no information with which to prefer one path over another (e.g. the shorter, and usually lower-latency, one).

This extension solves both with a single mechanism: an ordered list of **Hop IDs** that records the relays an advertisement has traversed.

## Why per-hop, not end-to-end
The Hop ID list is rewritten at every relay: a relay appends its own Hop ID before forwarding an advertisement downstream.
A relay therefore detects a loop by finding its own Hop ID already present in an incoming advertisement, and a subscriber compares path lengths using the list length.
Because the list is meaningful only within a single relay deployment, Hop IDs are scoped to that deployment and are not interpreted by endpoints outside it.


# Setup Negotiation
The Relay Hops extension is negotiated during the SETUP exchange as defined in {{moqt}} Section 10.3.
An endpoint indicates support by including the following Setup Option:

~~~
RELAY_HOPS Setup Option {
  Option Key (vi64) = 0x40B55
  Option Value Length (vi64) = 0
}
~~~

The extension applies to a single hop (one MOQT session) and is negotiated independently for each session; a relay MUST NOT assume that because one of its sessions negotiated Relay Hops, another did.

A relay that negotiated this extension on a downstream session SHOULD include the HOP_PATH parameter on every PUBLISH_NAMESPACE it sends on that session, and SHOULD honor an EXCLUDE_HOP parameter it receives in SUBSCRIBE_NAMESPACE.
An endpoint that did not negotiate the extension simply omits these parameters; per {{moqt}} an unknown Key-Value-Pair Type is ignored, so an advertisement forwarded into a non-supporting session loses its hop information gracefully.


# Hop IDs
A **Hop ID** is a variable-length integer that uniquely identifies a relay within a relay deployment (cluster).

- How Hop IDs are assigned is deployment-specific and out of scope for this document. A deployment MUST ensure each participating relay has a Hop ID that is unique within that deployment for as long as it is in use.
- The value `0` is reserved and means "unknown hop". It is used when an advertisement was bridged from a peer that does not support this extension (or an older protocol version), so that the hop count still reflects the true path length even when an intermediate identity is unavailable. A relay MUST NOT assign `0` to itself.

A relay MUST NOT reveal Hop IDs from one deployment to an unrelated deployment; see [Security Considerations](#security-considerations).


# HOP_PATH Parameter
The HOP_PATH parameter carries the ordered list of Hop IDs that an advertisement has traversed, from the origin toward the receiver.
It is a Key-Value-Pair (see {{moqt}} Section 2.5) carried in the Parameters of a PUBLISH_NAMESPACE message ({{moqt}} Section 10.15).

Because the value is a variable-length byte string, HOP_PATH uses an odd Type so that it is length-prefixed:

~~~
HOP_PATH Parameter {
  Type (vi64) = 0x40B57
  Length (vi64)
  Hop ID (vi64) ...
}
~~~

**Hop ID**:
Zero or more Hop IDs, ordered from the origin-most relay to the relay immediately upstream of the receiver.
The number of entries is determined by consuming Hop IDs until `Length` bytes have been read; a receiver MUST close the session with a PROTOCOL_VIOLATION if the entries do not exactly fill `Length`.
An empty value (Length 0) means the advertisement has not yet traversed any relay that assigns Hop IDs — typically because the sender is the origin publisher.

## Relay Behavior
When a relay forwards a namespace advertisement downstream on a session that negotiated this extension, it MUST append its own Hop ID to the HOP_PATH it received (creating an empty HOP_PATH first if the upstream advertisement carried none).
The relay's own Hop ID is therefore always the last entry of the list it sends.

When a relay receives a namespace advertisement on a session that negotiated this extension, it MUST inspect the HOP_PATH:

- If its own Hop ID already appears in the list, the advertisement has looped. The relay MUST NOT forward it and SHOULD drop it.
- Otherwise the relay MAY forward it downstream, appending its own Hop ID as described above.

When bridging from a peer that did not negotiate this extension, a relay MAY synthesize a single leading `0` ("unknown hop") entry to preserve an approximate path length, or MAY omit it; this choice is deployment-specific.

## Path Selection
A relay or subscriber that receives advertisements for the same namespace over multiple sessions MAY use the length of the HOP_PATH list as a tiebreaker, preferring the advertisement with the fewest hops (usually the lowest-latency path).
This is advisory: the receiver MAY apply additional local policy (e.g. measured RTT or administrative preference) and is not required to prefer the shortest path.

A publisher (or relay acting as one) SHOULD advertise only the single best path it currently knows for each namespace.
If the best path changes — for example after a relay failover — the publisher MAY re-advertise the namespace; the new advertisement, carrying an updated HOP_PATH, replaces the prior one per the namespace-advertisement semantics of {{moqt}}.


# EXCLUDE_HOP Parameter
The EXCLUDE_HOP parameter lets a downstream subscriber tell an upstream relay to suppress advertisements that have already passed through a given relay.
A relay in a cluster uses it to prevent the upstream from sending back an advertisement that the downstream originated, the most common source of a two-hop loop.

It is a Key-Value-Pair carried in the Parameters of a SUBSCRIBE_NAMESPACE message ({{moqt}} Section 10.18).
Because the value is a list, EXCLUDE_HOP uses an odd Type so that it is length-prefixed:

~~~
EXCLUDE_HOP Parameter {
  Type (vi64) = 0x40B59
  Length (vi64)
  Hop ID (vi64) ...
}
~~~

**Hop ID**:
One or more Hop IDs to exclude, in any order.
The number of entries is determined by consuming Hop IDs until `Length` bytes have been read; a receiver MUST close the session with a PROTOCOL_VIOLATION if the entries do not exactly fill `Length`, or if `Length` is 0.

A relay that receives a SUBSCRIBE_NAMESPACE carrying EXCLUDE_HOP SHOULD NOT send, on that session, any PUBLISH_NAMESPACE whose HOP_PATH contains any of the listed Hop IDs (including the implicit final entry the relay would itself append).
The exclusion is scoped to the namespace subscription it accompanies.

A relay that receives EXCLUDE_HOP without having negotiated the Relay Hops extension ignores it as an unknown parameter, which is the safe default (it simply does not perform the exclusion).


# Security Considerations
Hop IDs describe the internal topology of a relay deployment.
A relay MUST treat Hop IDs as private to the deployment: it MUST NOT forward HOP_PATH or EXCLUDE_HOP across a trust boundary (for example, to a subscriber outside the operator's own relay cluster), and SHOULD strip the HOP_PATH parameter before forwarding an advertisement to such a peer.
Leaking Hop IDs could reveal cluster size, topology, or failover state to an untrusted party.

A malicious upstream could forge a HOP_PATH to influence a downstream's path selection (e.g. claiming a short path it cannot actually deliver).
Path selection using HOP_PATH is therefore advisory only; a receiver SHOULD corroborate it with locally measured signals (e.g. RTT) before relying on it, and MUST NOT make security decisions based on Hop IDs.

A malicious subscriber could supply a large EXCLUDE_HOP list to consume relay resources.
Implementations SHOULD bound the number of excluded Hop IDs they will accept and MAY reject a SUBSCRIBE_NAMESPACE whose EXCLUDE_HOP list exceeds that bound.


# IANA Considerations

This document requests the following registrations.
High, distinctive values are requested to avoid the low ranges reserved by {{moqt}} and to minimize collisions with provisional registrations by other extensions; they also avoid the greasing pattern (`0x7f * N + 0x9D`).
The two parameter Types are odd so that each is length-prefixed (see {{moqt}} Section 2.5).

## MoQ Setup Options

This document requests a registration in the "MoQ Setup Options" registry ({{moqt}} Section 15.4), whose policy is Specification Required.

| Value   | Name       | Reference     |
|:--------|:-----------|:--------------|
| 0x40B55 | RELAY_HOPS | This Document |

## MoQ Key-Value-Pair Types

This document requests registrations in the "MoQ Key-Value-Pair Types" registry ({{moqt}} Section 15), used for message parameters and object/track properties.

| Value   | Name        | Carried In         | Reference     |
|:--------|:------------|:-------------------|:--------------|
| 0x40B57 | HOP_PATH    | PUBLISH_NAMESPACE  | This Document |
| 0x40B59 | EXCLUDE_HOP | SUBSCRIBE_NAMESPACE| This Document |


--- back

# Acknowledgments
{:numbered="false"}

This document was drafted with the assistance of Claude, an AI assistant by Anthropic.
