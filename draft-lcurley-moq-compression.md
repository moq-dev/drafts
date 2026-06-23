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
  RFC8878:

informative:
  RFC7692:

--- abstract

This document defines a payload compression extension for MoQ Transport {{moqt}}.
A track-level Compression property names the algorithm the original publisher used for a track's object payloads.
Endpoints advertise the algorithms they can decode during SETUP, and a payload is compressed on a hop only when the receiver supports the named algorithm; otherwise it is sent verbatim.
Compression is scoped to a subgroup: the object payloads of a subgroup form one compressed stream, sliced back into the individual object payloads so the object framing stays in the clear and the decompressed bytes — the actual objects — are unchanged end to end.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Introduction
{{moqt}} makes the original publisher "solely responsible for the content of the object payload ... including the underlying encoding, compression, any end-to-end encryption, or authentication" ({{moqt}} Section 2.1).
For media this is the right layering: already-compressed codecs (H.264, Opus, AV1) gain nothing from a second compression pass.

But MoQ also carries non-media tracks — JSON, text, telemetry, captions, uncompressed binary structures — where the payloads are highly compressible and where end-to-end encryption is often not in use.
For these tracks there is no standard, transport-visible way to compress payloads, so each application reinvents it, and relays cannot help.

Like HTTP Transfer-Encoding, the on-wire compression is a hop-by-hop optimization: it does not conceptually change the object payload — the decompressed bytes *are* the object — it only changes how those bytes are carried over a single hop.

- **Publisher names, hops apply**: the COMPRESSION track property names the algorithm the original publisher used; it is carried end to end and forwarded unchanged. A payload is compressed on a hop only when the receiver advertised that algorithm; otherwise it travels verbatim. Each hop's behavior is fixed by the publisher's algorithm and that hop's negotiation, with no per-object signal.
- **Per subgroup, sliced into objects**: within a subgroup the object payloads form one compressed stream, flushed at each object boundary so every object still carries its own payload slice, while the object headers and framing stay in the clear. This keeps the subgroup — already one ordered, reliable stream — as the unit of compression, and lets relays and caches store payloads compressed and re-frame them without recompressing.


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
An endpoint that includes this option MUST list `deflate` (1); the identifier `none` (0) MUST NOT be listed (it requires no negotiation).
An endpoint that does not support the extension omits the option.

A sender MUST NOT compress with an algorithm the receiver did not advertise, and MUST NOT compress before it has received the receiver's COMPRESSION option.
This makes the on-wire state unambiguous with no per-object signaling: a receiver decompresses a track's object payloads **if and only if** the COMPRESSION property names a non-`none` algorithm and the receiver advertised that algorithm in its own SETUP.
In every other case — the property absent or `none`, the extension not negotiated, or the algorithm not advertised by the receiver — the sender was not permitted to compress, so the receiver treats the payloads as verbatim.


# COMPRESSION Track Property
The COMPRESSION property names the algorithm the original publisher applied to a track's object payloads.
It is a track-level Key-Value-Pair carried with the track's properties (see {{moqt}} Section 2.5 and Section 12), set by the original publisher and forwarded unchanged by relays.
Because the value is a single integer, COMPRESSION uses an even Type so the value is a bare varint:

~~~
COMPRESSION Track Property {
  Type (vi64) = 0xC03D0
  Value (vi64)  ; Algorithm identifier
}
~~~

**Value**:
The Algorithm identifier the publisher used for this track's payloads (see [Compression Algorithms](#compression-algorithms)).
The absence of the property, or a value of `none` (0), means the track is uncompressed and its payloads are always transmitted verbatim.
The publisher MUST choose an algorithm that its peer advertised in the [COMPRESSION Setup Option](#setup-negotiation); since `deflate` is mandatory to implement, it is always a safe choice.

The property is fixed for the lifetime of the track and MUST NOT change.
A relay MUST forward it unchanged on every hop, including a hop that has not negotiated the extension: there it is simply an ignored unknown Key-Value-Pair, but forwarding it lets a further-downstream hop that does negotiate the extension still act on the publisher's algorithm.

Whether a payload is actually compressed is decided per hop:

- On a hop where the receiver advertised the property's algorithm, each non-empty object payload is compressed with that algorithm, and the receiver decompresses it.
- On any other hop — the extension not negotiated, or the receiver did not advertise that algorithm — payloads are sent verbatim, and the receiver treats them as such.

Compression applies to the object payload only; object headers, properties, and message framing are never compressed.
An empty payload (size 0) MUST NOT be compressed and remains empty on the wire.

A publisher SHOULD set COMPRESSION only for payload types that benefit from it.
Already-compressed media SHOULD omit it (or use `none`).


# Compression {#compression}
Compression is **scoped to a subgroup**.
Within a subgroup the object payloads form a single compressed stream in the algorithm named by the [COMPRESSION property](#compression-track-property), reset at each subgroup boundary.
The stream's output is partitioned at object boundaries: the compressor flushes at the end of each object so that object's slice is exactly the bytes carried as its payload, and the payload length in the object header gives the on-wire (compressed) slice size.
Both algorithms provide a window-retaining flush (DEFLATE's sync flush; Zstandard's `ZSTD_e_flush`), so later objects in a subgroup reuse the compression context and retain cross-object redundancy.

A receiver maintains a single decoder per subgroup, reset at each subgroup boundary, and feeds each object's payload through it in order: the first object of a subgroup starts the decoder fresh — so a receiver joining at a group boundary needs nothing earlier — while later objects build on it.
There is no shared state between subgroups; an empty payload contributes nothing to the stream.
An object delivered as a datagram is a single-object stream, compressed on its own.

Because the object framing already delimits each slice, an algorithm's own redundant boundary and container bytes are omitted: for `deflate`, the trailing four `00 00 FF FF` bytes a sync flush emits are removed from each payload and the decoder re-inserts them (as in {{RFC7692}}); for `zstd`, the per-subgroup stream uses the magicless frame format and omits the content checksum.

Leaving the framing uncompressed is deliberate.
A relay or cache can hold the object payloads compressed in memory and forward them without inflating, and can re-frame a subgroup — for example to bridge a transport version that changes the subgroup or object headers — without touching the compressed payloads.

## Compression Algorithms {#compression-algorithms}
This document defines the following algorithms.

| ID | Name    | Requirement | Description                                             |
|---:|:--------|:------------|:--------------------------------------------------------|
| 0  | none    | —           | Verbatim; the absence of compression. Never advertised. |
| 1  | deflate | mandatory   | Raw DEFLATE {{RFC1951}}, with no zlib or gzip framing.   |
| 2  | zstd    | optional    | Zstandard {{RFC8878}}.                                   |

Every endpoint that advertises this extension MUST implement `deflate`, so the publisher always has a safe choice; `zstd` is optional.
Further algorithms MAY be registered (see [IANA Considerations](#iana-considerations)).


# Relay Behavior
A relay forwards the COMPRESSION track property unchanged — it is the publisher's end-to-end signal — and applies compression independently on each hop, driven by each hop's negotiation rather than by its own initiative; a relay does not compress a track the publisher did not mark.

On its upstream subscription the relay receives each subgroup compressed with the property's algorithm (if it advertised that algorithm) or verbatim, and decompresses as needed.
On a downstream subscription that advertised the property's algorithm, it sends each subgroup compressed with that algorithm (recompressing as needed); on one that did not, it sends the subgroup verbatim.
A relay MUST NOT recompress with an algorithm other than the one the property names, because the property tells the receiver how to decode and a relay MUST NOT rewrite it.
In every case the decompressed bytes delivered to the application MUST be identical to what the origin published, and a relay or generic library MUST NOT inspect or modify the decompressed contents unless otherwise negotiated.

Open issue:
because the COMPRESSION property both names the algorithm and is immutable, a downstream that supports a *different* algorithm than the publisher chose (for example only `deflate` when the publisher used `zstd`) receives the payloads verbatim rather than transcoded — a relay cannot offer it the algorithm it does support without rewriting the property, which {{moqt}} forbids.
Whether to relax this — by permitting a relay to rewrite this property for a downstream subscription, or by carrying the per-hop algorithm as transport metadata rather than an end-to-end track property — is an open question for the working group.


# Security Considerations
Compressing data that mixes attacker-controlled and secret content can leak the secret through compressed size, as in the CRIME and BREACH attacks.
A publisher MUST NOT set COMPRESSION on a track whose object payloads combine secret material with attacker-influenced material.
Because compression is scoped to a subgroup, the exposure is bounded to within a single subgroup — which may combine several objects, a wider window than a single object — but it is not eliminated.

A malicious sender could emit a small compressed payload that decompresses to a very large buffer (a "decompression bomb").
Because compression is subgroup-scoped, a receiver MUST bound the cumulative decompressed size of a subgroup — not merely each object's payload, since many small payloads can otherwise accumulate without limit. If the bound is exceeded it MUST reset the affected stream (rather than allocate unbounded memory) and MAY close the session with a PROTOCOL_VIOLATION if it considers the peer abusive; the reset is stream-scoped so a single bad subgroup does not tear down unrelated subscriptions.

Compression is orthogonal to {{moqt}} end-to-end encryption: an encrypted payload is effectively incompressible, so a publisher using end-to-end encryption SHOULD omit COMPRESSION (or use `none`).


# IANA Considerations

This document requests the following registrations.
High, distinctive values are requested to avoid the low ranges reserved by {{moqt}} and to minimize collisions with provisional registrations by other extensions; they also avoid the greasing pattern (`0x7f * N + 0x9D`).
Each Type is even so that its value is a bare varint with no length prefix (see {{moqt}} Section 2.5).

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
| 2  | zstd    | This Document |


--- back

# Acknowledgments
{:numbered="false"}

This document was drafted with the assistance of Claude, an AI assistant by Anthropic.
