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
A track-level Compression property declares that every object payload on the track is compressed with a negotiated algorithm, applied independently per object so objects remain individually decodable.
Compression is negotiated per hop during SETUP, allowing a relay to transcode (including bridging endpoints that do not support the extension) while preserving the decompressed bytes exactly.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Introduction
{{moqt}} makes the original publisher "solely responsible for the content of the object payload ... including the underlying encoding, compression, any end-to-end encryption, or authentication" ({{moqt}} Section 2.1).
For media this is the right layering: already-compressed codecs (H.264, Opus, AV1) gain nothing from a second compression pass.

But MoQ also carries non-media tracks — JSON, text, telemetry, captions, uncompressed binary structures — where the payloads are highly compressible and where end-to-end encryption is often not in use.
For these tracks there is no standard, transport-visible way to compress payloads, so each application reinvents it, and relays cannot help.

This extension defines a per-track, per-object compression scheme negotiated at the transport layer:

- **Per object, independently**: each object payload is an independent compressed stream with no shared dictionary or state between objects. This keeps every object individually decodable and avoids head-of-line decoding within a group.
- **Per hop, negotiated**: compression is negotiated during SETUP for each session, so a relay can decompress, recompress with a different algorithm, or bridge to a peer that does not support the extension — as long as the decompressed bytes are identical end to end.


# Setup Negotiation
The Payload Compression extension is negotiated during the SETUP exchange as defined in {{moqt}} Section 10.3.
Unlike a purely additive property, compression MUST be negotiated: a receiver that does not understand the algorithm would otherwise pass the compressed bytes to the application as if they were plaintext.

An endpoint advertises the algorithms it can decompress (when acting as a subscriber) and produce (when acting as a publisher) by including the following Setup Option:

~~~
COMPRESSION Setup Option {
  Option Key (vi64) = 0xC03DE
  Option Value Length (vi64)
  Algorithm (vi64) ...
}
~~~

**Algorithm**:
One or more Algorithm identifiers (see [Compression Algorithms](#compression-algorithms)) that the sender supports, each a varint, filling the Option Value.
The identifier `none` (0) MUST NOT be listed (it requires no negotiation).

A publisher MUST NOT compress a track with an algorithm the peer did not advertise in its SETUP.
If a receiver encounters a COMPRESSION track property naming an algorithm it did not advertise — for example because an object arrived before SETUP, or a peer misbehaved — it MUST reset the affected subscription/fetch with a PROTOCOL_VIOLATION rather than deliver the payload uninterpreted.


# COMPRESSION Track Property
The COMPRESSION property declares the algorithm applied to every object payload on a track.
It is a track-level Key-Value-Pair carried with the track's properties (see {{moqt}} Section 2.5 and Section 12).
Because the value is a single integer, COMPRESSION uses an even Type so the value is a bare varint:

~~~
COMPRESSION Track Property {
  Type (vi64) = 0xC03D0
  Value (vi64)  ; Algorithm identifier
}
~~~

**Value**:
The Algorithm identifier applied to every object payload on the track.
The absence of the property, or a value of `none` (0), means payloads are transmitted verbatim.

The compression algorithm is fixed for the lifetime of the track and MUST NOT change.
Compression applies to the object payload only; object properties and message framing are never compressed.
An empty payload (size 0) MUST NOT be compressed and remains empty on the wire.

A publisher SHOULD enable compression only for payload types that benefit from it.
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
A relay MAY forward compressed payloads unchanged, decompress them, or recompress them with a different algorithm, provided the decompressed bytes delivered to the original-format consumer are identical to what the publisher produced.

A relay MAY transcode between algorithms — including bridging a peer that supports this extension to one that does not (by decompressing to `none`), or bridging two peers that support different algorithm sets.
When it does, it rewrites the COMPRESSION track property accordingly on the downstream session.

A relay SHOULD NOT compress an originally-uncompressed payload unless there is a strong content signal that compression is beneficial (for example, a track name ending in `.json`), because it cannot otherwise predict whether compression will help or hurt.

A relay or generic library MUST NOT inspect or modify the decompressed contents unless otherwise negotiated; only recompression that preserves the decompressed bytes exactly is permitted.


# Security Considerations
Compressing data that mixes attacker-controlled and secret content in the same object can leak the secret through compressed size, as in the CRIME and BREACH attacks.
A publisher MUST NOT enable compression on a track whose object payloads combine secret material with attacker-influenced material.
Because compression here is per-object with no cross-object dictionary, the exposure is bounded to within a single object, but it is not eliminated.

A malicious publisher could send a small compressed payload that decompresses to a very large buffer (a "decompression bomb").
A receiver MUST bound the size of a decompressed object payload and MUST reset the stream with a PROTOCOL_VIOLATION (or an application error) if the bound is exceeded, rather than allocating unbounded memory.

Compression is orthogonal to {{moqt}} end-to-end encryption: an encrypted payload is effectively incompressible, so a publisher using end-to-end encryption SHOULD use `none`.


# IANA Considerations

This document requests the following registrations.
High, distinctive values are requested to avoid the low ranges reserved by {{moqt}} and to minimize collisions with provisional registrations by other extensions; they also avoid the greasing pattern (`0x7f * N + 0x9D`).
The property Type is even so that its value is a bare varint with no length prefix (see {{moqt}} Section 2.5).

## MOQT Setup Options

This document requests a registration in the "MOQT Setup Options" registry ({{moqt}} Section 15.4), whose policy is Specification Required.

| Value   | Name        | Reference     |
|:--------|:------------|:--------------|
| 0xC03DE | COMPRESSION | This Document |

## MOQT Properties

This document requests a registration in the "MOQT Properties" registry ({{moqt}} Section 15.8), used for object and track properties.

| Value   | Name        | Scope | Reference     |
|:--------|:------------|:------|:--------------|
| 0xC03D0 | COMPRESSION | Track | This Document |

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
