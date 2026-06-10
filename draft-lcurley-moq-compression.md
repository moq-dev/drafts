---
title: "MoQ Payload Compression Extension"
abbrev: "moq-compression"
category: info

docname: draft-lcurley-moq-compression-latest
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
  RFC1951:

informative:

--- abstract

This document defines a payload compression extension for MoQ Transport {{moqt}}.
Compression is a hop-by-hop wire optimization, analogous to HTTP Transfer-Encoding: the endpoint serving a subscription MAY compress object payloads with an algorithm the receiver advertised, signaling the algorithm per subscription. Each object is compressed independently so objects remain individually decodable.
Because it is hop-by-hop, the origin publisher need not participate: any relay can compress toward a downstream that supports it, or decompress toward one that does not, and the decompressed bytes — the actual object — are unchanged end to end.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Introduction
{{moqt}} makes the original publisher "solely responsible for the content of the object payload ... including the underlying encoding, compression, any end-to-end encryption, or authentication" ({{moqt}} Section 2.1).
For media this is the right layering: already-compressed codecs (H.264, Opus, AV1) gain nothing from a second compression pass.

But MoQ also carries non-media tracks — JSON, text, telemetry, captions, uncompressed binary structures — where the payloads are highly compressible and where end-to-end encryption is often not in use.
For these tracks there is no standard, transport-visible way to compress payloads, so each application reinvents it, and relays cannot help.

This extension defines a hop-by-hop, per-object compression scheme negotiated at the transport layer.
It is a wire optimization, analogous to HTTP Transfer-Encoding (a hop-by-hop encoding) rather than HTTP Content-Encoding (an end-to-end one): it does not conceptually change the object payload — the decompressed bytes *are* the object — it only changes how those bytes are carried over a single hop.

- **Hop-by-hop, not end-to-end**: compression is negotiated during SETUP for each session and applied independently on each hop by whichever endpoint is serving the subscription. The origin publisher need not opt in; a relay can compress toward a downstream that supports it and decompress toward one that does not.
- **Per object, independently**: each object payload is an independent compressed stream with no shared dictionary or state between objects. This keeps every object individually decodable and avoids head-of-line decoding within a group.


# Setup Negotiation
The Payload Compression extension is negotiated during the SETUP exchange as defined in {{moqt}} Section 10.3.
Unlike a purely additive property, compression MUST be negotiated: a receiver that does not understand the algorithm would otherwise pass the compressed bytes to the application as if they were plaintext.

Each endpoint advertises the algorithms it can decompress by including the following Setup Option:

~~~
COMPRESSION Setup Option {
  Option Key (vi64) = 0xC03DE
  Option Value Length (vi64)
  Algorithm (vi64) ...
}
~~~

**Algorithm**:
One or more Algorithm identifiers (see [Compression Algorithms](#compression-algorithms)) that the sender can decompress, each a varint, filling the Option Value.
The identifier `none` (0) MUST NOT be listed (it requires no negotiation).

A sender MUST NOT compress with an algorithm the receiver did not advertise in its SETUP.
If a receiver is told — via the COMPRESSION subscription property below — that objects use an algorithm it did not advertise (for example because data arrived before SETUP, or a peer misbehaved), it MUST reset the affected subscription/fetch with a PROTOCOL_VIOLATION rather than deliver the payload uninterpreted.


# COMPRESSION Subscription Property
Compression is signaled per subscription by the endpoint serving it.
The COMPRESSION property declares the algorithm applied to the object payloads delivered for a given subscription (or fetch).
It is a Key-Value-Pair (see {{moqt}} Section 2.5) carried as a parameter of SUBSCRIBE_OK (and the corresponding FETCH_OK), sent by the publisher or relay that serves the request to the receiver.
Because the value is a single integer, COMPRESSION uses an even Type so the value is a bare varint:

~~~
COMPRESSION Subscription Property {
  Type (vi64) = 0xC03D0
  Value (vi64)  ; Algorithm identifier
}
~~~

**Value**:
The Algorithm identifier applied to every object payload delivered for this subscription.
The absence of the property, or a value of `none` (0), means payloads are transmitted verbatim.

The algorithm is fixed for the lifetime of the subscription; to change it, the receiver re-subscribes.
Compression applies to the object payload only; object properties and message framing are never compressed.
An empty payload (size 0) MUST NOT be compressed and remains empty on the wire.

A sender SHOULD enable compression only for payload types that benefit from it.
Already-compressed media SHOULD use `none`.


# Compression Algorithms {#compression-algorithms}
This document defines the following algorithms.
Further algorithms MAY be registered (see [IANA Considerations](#iana-considerations)).

| ID | Name    | Description                                              |
|---:|:--------|:--------------------------------------------------------|
| 0  | none    | Payloads are transmitted verbatim. The default.        |
| 1  | deflate | Raw DEFLATE {{RFC1951}}, with no zlib or gzip framing.  |

For `deflate`, each object payload is an independent raw DEFLATE stream.
There is no shared dictionary or state between objects, so each object decompresses on its own.


# Relay Behavior
Because compression is hop-by-hop, a relay handles each hop independently.
On its upstream subscription it receives objects compressed with whatever algorithm that hop signaled (or none) and decompresses them as needed.
On each downstream subscription it serves, it independently chooses whether to compress — using any algorithm that downstream advertised — and signals its choice with the COMPRESSION subscription property on that SUBSCRIBE_OK / FETCH_OK.

A relay MAY therefore bridge a compressing hop to a non-supporting one (by decompressing), compress an originally-uncompressed upstream toward a downstream that supports it, or recompress with a different algorithm.
In every case the decompressed bytes delivered to the application MUST be identical to what the origin published.

A relay SHOULD NOT compress an originally-uncompressed payload unless there is a strong content signal that compression is beneficial (for example, a track name ending in `.json`), because it cannot otherwise predict whether compression will help or hurt.

A relay or generic library MUST NOT inspect or modify the decompressed contents unless otherwise negotiated; only recompression that preserves the decompressed bytes exactly is permitted.


# Security Considerations
Compressing data that mixes attacker-controlled and secret content in the same object can leak the secret through compressed size, as in the CRIME and BREACH attacks.
A sender MUST NOT enable compression for payloads that combine secret material with attacker-influenced material.
Because compression here is per-object with no cross-object dictionary, the exposure is bounded to within a single object, but it is not eliminated.

A malicious sender could emit a small compressed payload that decompresses to a very large buffer (a "decompression bomb").
A receiver MUST bound the size of a decompressed object payload and MUST reset the stream with a PROTOCOL_VIOLATION (or an application error) if the bound is exceeded, rather than allocating unbounded memory.

Compression is orthogonal to {{moqt}} end-to-end encryption: an encrypted payload is effectively incompressible, so a sender SHOULD NOT compress an end-to-end-encrypted payload.


# IANA Considerations

This document requests the following registrations.
High, distinctive values are requested to avoid the low ranges reserved by {{moqt}} and to minimize collisions with provisional registrations by other extensions; they also avoid the greasing pattern (`0x7f * N + 0x9D`).
The parameter Type is even so that its value is a bare varint with no length prefix (see {{moqt}} Section 2.5).

## MOQT Setup Options

This document requests a registration in the "MOQT Setup Options" registry ({{moqt}} Section 15.4), whose policy is Specification Required.

| Value   | Name        | Reference     |
|:--------|:------------|:--------------|
| 0xC03DE | COMPRESSION | This Document |

## MOQT Message Parameters

This document requests a registration in the "MOQT Message Parameters" registry ({{moqt}} Section 15.7).
COMPRESSION is a subscription property carried in SUBSCRIBE_OK and FETCH_OK.

| Value   | Name        | Carried In            | Reference     |
|:--------|:------------|:----------------------|:--------------|
| 0xC03D0 | COMPRESSION | SUBSCRIBE_OK, FETCH_OK | This Document |

## MOQT Compression Algorithms

This document requests a new "MOQT Compression Algorithms" registry, with a registration policy of Specification Required.
The initial contents are:

| ID | Name    | Reference     |
|---:|:--------|:--------------|
| 0  | none    | This Document |
| 1  | deflate | This Document |


--- back

# Acknowledgments
{:numbered="false"}

This document was drafted with the assistance of Claude, an AI assistant by Anthropic.
