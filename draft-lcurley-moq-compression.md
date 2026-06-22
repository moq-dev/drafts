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
A track-level Compression property is a boolean hint by which the original publisher signals that a track's object payloads are worth compressing.
The algorithm is negotiated independently on each hop, and compression is applied per hop: an object is compressed only on a hop that has negotiated the extension and a shared algorithm, and is sent verbatim otherwise.
Each object is compressed independently so objects remain individually decodable, an object can opt out when compression would not help, and the decompressed bytes — the actual object — are unchanged end to end.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Introduction
{{moqt}} makes the original publisher "solely responsible for the content of the object payload ... including the underlying encoding, compression, any end-to-end encryption, or authentication" ({{moqt}} Section 2.1).
For media this is the right layering: already-compressed codecs (H.264, Opus, AV1) gain nothing from a second compression pass.

But MoQ also carries non-media tracks — JSON, text, telemetry, captions, uncompressed binary structures — where the payloads are highly compressible and where end-to-end encryption is often not in use.
For these tracks there is no standard, transport-visible way to compress payloads, so each application reinvents it, and relays cannot help.

Like HTTP Transfer-Encoding, the on-wire compression is a hop-by-hop optimization: it does not conceptually change the object payload — the decompressed bytes *are* the object — it only changes how those bytes are carried over a single hop.
What this extension adds on top is an end-to-end *signal*: a boolean track property by which the original publisher marks the content as worth compressing. The signal travels end to end; the choice of algorithm and the compression itself happen per hop.

- **Publisher signals, hops apply**: the COMPRESSION track property is set by the original publisher and carried end to end, but a payload is only compressed on a hop that negotiated the extension and a shared algorithm. Where the extension is not negotiated, the same payload travels verbatim.
- **Per object, independently**: each object payload is an independent compressed stream with no shared dictionary or state between objects. This keeps every object individually decodable, avoids head-of-line decoding within a group, and lets an individual object opt out of compression when it would not benefit.


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
They are listed in the sender's order of preference, most-preferred first.
The identifier `none` (0) MUST NOT be listed (it requires no negotiation).

A sender MUST NOT compress with an algorithm the receiver did not advertise in its SETUP.
The negotiated algorithm for a hop — the **hop default** — is the first algorithm in the receiver's advertised list that the sender can also produce; if the lists do not intersect, the hop has no default and every payload travels verbatim.
This keeps the common case free of per-object signaling: where the COMPRESSION track property is present and the hop has a default, a receiver decompresses each object with the hop default unless that object carries a [per-object override](#per-object-override) naming a different algorithm (in particular `none`, to send it verbatim). Where the property is absent, the extension is not negotiated, or the algorithm lists do not intersect, the sender was not permitted to compress, so every payload is verbatim.


# COMPRESSION Track Property
The COMPRESSION property is the original publisher's signal that a track's object payloads are worth compressing.
It is a track-level Key-Value-Pair carried with the track's properties (see {{moqt}} Section 2.5 and Section 12), set by the original publisher and forwarded unchanged by relays.
Because the value is a single integer, COMPRESSION uses an even Type so the value is a bare varint:

~~~
COMPRESSION Track Property {
  Type (vi64) = 0xC03D0
  Value (vi64)  ; boolean hint
}
~~~

**Value**:
A boolean hint: `1` means the track's payloads are worth compressing; `0`, or absence of the property, means they are not and are always transmitted verbatim.
Values greater than `1` are reserved for future use and MUST be treated as `1` by a receiver that does not understand them, so the hint stays additive.
The property names no algorithm: which algorithm is used, if any, is the per-hop negotiated [hop default](#setup-negotiation), overridable [per object](#per-object-override).

The property is fixed for the lifetime of the track and MUST NOT change.
A relay MUST forward it unchanged on every hop, including a hop that has not negotiated the extension: there it is simply an ignored unknown Key-Value-Pair, but forwarding it lets a further-downstream hop that does negotiate the extension still act on the publisher's signal.

Compression is enabled only by the combination of a non-zero hint and the extension being negotiated with a shared algorithm on a hop.
A publisher MUST NOT compress object payloads on a track that does not carry a non-zero COMPRESSION hint.

Whether a given object is actually compressed is decided per hop and per object:

- On a hop where the extension is negotiated and a [hop default](#setup-negotiation) exists, each non-empty object payload is compressed with the hop default and the receiver decompresses it — unless the object carries a [per-object override](#per-object-override), which names the algorithm actually used (including `none`, to send that object verbatim).
- On any other hop — the extension not negotiated, or the algorithm lists do not intersect — payloads are sent verbatim. The receiver either never sees the property (an ignored unknown Key-Value-Pair) or sees it but knows the sender was not permitted to compress for it, so it treats the payloads as verbatim either way.

A sender SHOULD send an object verbatim (via a `none` override) whenever the hop default would not make that object smaller — for example a small JSON merge-patch delta that DEFLATE would enlarge.
Compression applies to the object payload only; object properties and message framing are never compressed.
An empty payload (size 0) MUST NOT be compressed and remains empty on the wire; it needs no override.

A publisher SHOULD set COMPRESSION only for payload types that benefit from it.
Already-compressed media SHOULD omit it (or use `0`).


# Per-Object Override {#per-object-override}
The COMPRESSION_ALGORITHM property is an optional object-level Key-Value-Pair that overrides, for a single object, the algorithm a hop would otherwise apply.
Because the value is a single integer, it uses an even Type so the value is a bare varint:

~~~
COMPRESSION_ALGORITHM Object Property {
  Type (vi64) = 0xC03D2
  Value (vi64)  ; Algorithm identifier
}
~~~

**Value**:
The [algorithm](#compression-algorithms) actually used for this object's payload on this hop.
`none` (0) means the object is carried verbatim; any other identifier names the algorithm whose output the payload is.
A sender MUST NOT name an algorithm the receiver did not advertise in its SETUP.

The property is meaningful only on a hop that has a [hop default](#setup-negotiation); where present it replaces the hop default for that object alone, and elsewhere objects are always verbatim and any COMPRESSION_ALGORITHM property MUST be ignored.
Unlike the boolean COMPRESSION hint, it is not the publisher's end-to-end signal: because it records what a hop actually did, a relay rewrites or removes it to reflect what it did on each downstream hop, exactly as it does for the payload bytes.
Its typical use is a `none` override that keeps an incompressible or tiny object verbatim while the rest of the track is compressed.


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
A relay forwards the boolean COMPRESSION track property unchanged — it is the publisher's end-to-end signal — and applies compression independently on each hop.

On its upstream subscription, the relay receives each object compressed with that hop's default unless the object carried a per-object override; it reads the [COMPRESSION_ALGORITHM](#per-object-override) property, or the hop default in its absence, to decompress as needed.
On each downstream subscription the relay serves, it compresses each object with that downstream's hop default when one exists and sends objects verbatim otherwise, rewriting or removing the per-object COMPRESSION_ALGORITHM property to reflect what it actually did on that hop.

Compression is thus driven by the publisher's hint and each hop's negotiation, not by the relay's own initiative: a relay does not compress a track the publisher did not mark.
In every case the decompressed bytes delivered to the application MUST be identical to what the origin published.

A relay or generic library MUST NOT inspect or modify the decompressed contents unless otherwise negotiated; only recompression that preserves the decompressed bytes exactly is permitted.


# Security Considerations
Compressing data that mixes attacker-controlled and secret content in the same object can leak the secret through compressed size, as in the CRIME and BREACH attacks.
A publisher MUST NOT set a non-zero COMPRESSION hint on a track whose object payloads combine secret material with attacker-influenced material.
Because compression here is per-object with no cross-object dictionary, the exposure is bounded to within a single object, but it is not eliminated.

A malicious sender could emit a small compressed payload that decompresses to a very large buffer (a "decompression bomb").
A receiver MUST bound the size of a decompressed object payload. If the bound is exceeded it MUST reset the affected Subscribe/Fetch stream (rather than allocate unbounded memory) and MAY close the session with a PROTOCOL_VIOLATION if it considers the peer abusive; the reset is stream-scoped so a single bad object does not tear down unrelated subscriptions.

Compression is orthogonal to {{moqt}} end-to-end encryption: an encrypted payload is effectively incompressible, so a publisher using end-to-end encryption SHOULD omit COMPRESSION (or use `0`).


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

This document requests registrations in the "MOQT Properties" registry ({{moqt}} Section 15.8), used for object and track properties.

| Value   | Name                  | Scope  | Reference     |
|:--------|:----------------------|:-------|:--------------|
| 0xC03D0 | COMPRESSION           | Track  | This Document |
| 0xC03D2 | COMPRESSION_ALGORITHM | Object | This Document |

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
