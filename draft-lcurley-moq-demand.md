---
title: "MoQ Demand Extension"
abbrev: "moq-demand"
category: info

docname: draft-lcurley-moq-demand-latest
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

This document defines a SUBSCRIBE_DEMAND message for MoQ Transport {{moqt}}: fire-and-forget feedback that a subscriber reports about a subscription describing the downstream demand for it.
It carries how many subscribers a subscription represents — as a pair of cumulative counts whose difference telescopes up the relay fan-out tree, letting a publisher learn its total audience across any number of hops — and an optional Group Request asking the publisher to produce a new group once the subscriber has fallen behind.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Introduction
A publisher in {{moqt}} often wants to know the demand for a Track: how many subscribers are receiving it, and whether any of them needs a fresh group to make progress.
Both are straightforward when a subscriber connects directly, but {{moqt}} is designed around relays: a relay aggregates many downstream subscriptions for the same Track into a single upstream subscription toward the origin (its "fan-out" tree).
The origin sees one upstream subscription per relay, not the individual subscribers behind it, so it can neither count its true audience nor learn what those subscribers need without out-of-band coordination.

This document defines a SUBSCRIBE_DEMAND message that reports this demand back up the subscription path.
It carries two kinds of information, each chosen so that it aggregates cheaply up the fan-out tree:

- **Audience size**, as a pair of cumulative counts, `Subscriptions Created` and `Subscriptions Closed`. A relay reports the **sum** of each across the downstream subscriptions it serves, so the difference — the current number of subscribers — telescopes for free. At the origin, the demand on each upstream subscription is the total number of subscribers reachable through that relay, transitively, across any number of hops.
- A **Group Request**, the minimum group a subscriber wants the publisher to produce. A relay reports the **maximum** across its downstreams (less any it can already satisfy from cache). It is expressed as a *level* — "I want a group at least this new" — not a one-shot trigger, so it is idempotent and deduplicates naturally as it aggregates.

Demand changes as subscribers join, leave, and fall behind, so SUBSCRIBE_DEMAND is sent repeatedly over the life of a subscription.
It is a dedicated, fire-and-forget message rather than a parameter on REQUEST_UPDATE ({{moqt}} Section 10.9).
A REQUEST_UPDATE consumes a Request ID ({{moqt}} Section 10.1) and obliges the receiver to answer with a REQUEST_OK or REQUEST_ERROR ({{moqt}} Section 10.9) — a request/response transaction whose purpose is to *modify* the subscription's delivery range or priority, which is a poor fit for feedback pushed repeatedly that changes neither.
SUBSCRIBE_DEMAND instead rides the subscription's existing request stream, consumes no Request ID, and elicits no response.


# Setup Negotiation
The Demand extension is negotiated during the SETUP exchange as defined in {{moqt}} Section 9.4.

Both endpoints indicate support by including the following Setup Option:

~~~
SUBSCRIBE_DEMAND Setup Option {
  Option Key (vi64) = 0xC0117
  Option Value Length (vi64) = 0
}
~~~

The extension is available on a hop only if both endpoints on that hop included this option.
The extension is negotiated independently on each hop: a relay MAY support it upstream but not downstream, or vice versa.

Negotiation is mandatory before the message is sent.
{{moqt}} (Section 10) requires an endpoint that receives an unknown control message type to close the session, so — unlike an optional parameter, which can be ignored — a SUBSCRIBE_DEMAND message cannot be sent speculatively.
An endpoint MUST NOT send SUBSCRIBE_DEMAND on a hop that did not negotiate this extension.


# SUBSCRIBE_DEMAND Message
This document defines a new control message, sent on a subscription's request stream ({{moqt}} Section 3.3) by the endpoint that opened it (the subscriber, which for an upstream subscription is the relay).

~~~
SUBSCRIBE_DEMAND Message {
  Type (vi64) = 0xC0117
  Length (16)
  Subscriptions Created (vi64)
  Subscriptions Closed (vi64)
  Group Request (vi64)
}
~~~

The message MUST NOT be the first message on the request stream; it follows the SUBSCRIBE ({{moqt}} Section 10.7) that opened the stream.
It consumes no Request ID ({{moqt}} Section 10.1), and the receiver MUST NOT respond to it.
A subscriber MAY send it any number of times over the life of the subscription to refresh the values.

**Subscriptions Created** and **Subscriptions Closed**:
Cumulative counts, over the life of this subscription, of the downstream subscriptions it represents that have been created and closed respectively.
The current demand — the number of subscribers presently receiving the Track through this subscription — is `Subscriptions Created - Subscriptions Closed`.
A leaf subscriber represents only itself: `Subscriptions Created` is `1` and `Subscriptions Closed` is `0`.
These are the defaults until a SUBSCRIBE_DEMAND is received, so a subscriber that represents only itself need not send the message.

**Group Request**:
The minimum group the subscriber wants the publisher to produce.
A value of `0` means no request: the publisher produces groups at its own cadence.
A non-zero value `N` requests that the publisher produce a group with Group ID at least `N - 1`; the offset by one keeps `0` available as "no request" while leaving Group ID `0` requestable.
See [Group Requests](#group-requests) for the semantics.


# Semantics

## Audience Size
The audience size is a reduction up the subscription tree.

A **leaf subscriber** (one that is not a relay) represents itself: `Subscriptions Created` of `1` and `Subscriptions Closed` of `0`, a demand of `1`.
It need not report this default.

A **relay** that aggregates one or more downstream subscriptions for a Track into a single upstream subscription reports, on that upstream subscription, the **sum** of the `Subscriptions Created` of its downstreams and the **sum** of their `Subscriptions Closed`, treating a downstream that has not reported as `1` created and `0` closed.
It increments `Subscriptions Created` as downstream subscriptions are created and `Subscriptions Closed` as they are closed, and SHOULD keep both counts non-decreasing over the upstream subscription's life — accounting a fully-departed downstream's outstanding demand (its last-reported `Created - Closed`) as newly closed — so that neither count moves backward when a downstream detaches.
When either sum changes, the relay sends a SUBSCRIBE_DEMAND message upstream with the new totals.

Because each relay reports the sum of its subtree, the difference telescopes: at the origin, `Created - Closed` on a given upstream subscription is the total number of leaf subscribers reachable through that subscription, across any number of relay hops.
A publisher reads its total audience for a Track as the sum of `Created - Closed` over the subscriptions it is serving.

Reporting the two counts separately, rather than a single current-demand gauge, lets a publisher distinguish *churn* from a *new arrival*: a rising `Subscriptions Created` means at least one new subscriber has joined, which a publisher MAY treat as an implicit [Group Request](#group-requests), since a newly-joined subscriber generally needs a fresh group (e.g. a keyframe) to begin decoding.

## Group Requests {#group-requests}
A subscriber raises `Group Request` to ask the publisher to start a new group once it has fallen too far behind the live edge to catch up — for example, after missing the group with ID `5` it requests `6` to jump to the next group rather than wait for it to be produced naturally.

The request is a **level**, not an edge: it names the minimum group the subscriber needs, and is satisfied the moment a group at or beyond it exists.
A publisher that has already produced a group with Group ID at or above the request takes no action — the request is already met.
This is the key difference from a one-shot "produce a new group now" signal: because the request is idempotent, it can be retransmitted, coalesced, and aggregated without a publisher producing one redundant group per copy it receives.

`Group Request` fans *in* at a relay as the **maximum** of its downstreams' requests, minus any the relay can already satisfy itself: a relay that holds a group at or beyond a downstream's request serves it from cache and does not propagate it; it forwards a request upstream only when it lacks a group at or beyond the highest value its downstreams want.
Once the publisher produces a group satisfying the highest request, every lower request is satisfied at once.

A publisher SHOULD honor a `Group Request` by producing a new group as soon as it can (subject to its own encoding constraints, such as a keyframe boundary), but MAY decline or defer it; the request does not override the publisher's control of its own Track.
`Group Request` is the only field of this message that affects delivery. The audience-size counts MUST NOT influence prioritization, caching, congestion response, or any other distribution decision beyond the optional new-group hint above.


# Rate Limiting
Subscriber churn can change the audience size rapidly, and at a busy relay each change would otherwise produce an upstream SUBSCRIBE_DEMAND message.

A relay SHOULD rate-limit SUBSCRIBE_DEMAND messages per subscription, coalescing audience-size changes that occur within a short window (on the order of a second) and then sending the latest values.
Because each message carries current values rather than deltas, a change that reverts within the window — a subscriber that joins and leaves, or leaves and returns — requires no upstream message at all.

A `Group Request` increase is latency-sensitive — the subscriber is stalled waiting for a group it can decode — and SHOULD be forwarded promptly rather than held for the audience-size window.
Because the message is independent of REQUEST_UPDATE, neither kind of update delays a genuine subscription change: delivery-affecting updates are forwarded according to {{moqt}} without regard to the demand window.


# Security Considerations
**Audience disclosure.**
`Subscriptions Created` and `Subscriptions Closed` disclose aggregate viewership to the publisher and to every relay on the path toward it.
For some applications the size of an audience is sensitive (for example, it can reveal the popularity or reach of content, or that an audience has dropped to zero).
Because the extension is negotiated per hop, an endpoint that considers this sensitive simply does not advertise the SUBSCRIBE_DEMAND Setup Option, and no demand is exchanged on that hop.

**Untrusted values.**
The values are supplied by the subscriber side and aggregated by intermediaries, none of which the publisher can fully trust.
A malicious or buggy subscriber can report inflated or deflated counts, and a malicious relay can report any sums regardless of its actual downstream subscriptions.
The audience-size counts are therefore advisory: an endpoint MUST NOT use one for any security-sensitive purpose — such as billing, admission control, rate limiting, or capacity planning that affects other subscribers — without independent verification.
A `Group Request` can at most ask the publisher to produce a group it could already have produced for any subscriber, and a publisher MAY decline it; honoring requests at an unbounded rate would let a subscriber drive group production, so a publisher SHOULD bound the rate at which it acts on requests.

**Churn amplification.**
A subscriber that rapidly joins and leaves, or repeatedly raises its `Group Request`, could attempt to amplify control traffic toward the origin.
The rate-limiting in [Rate Limiting](#rate-limiting) bounds the audience-size case, and the idempotent, level-based `Group Request` collapses repeated identical requests into a single upstream value and a single produced group.
Because the message consumes no Request ID and elicits no response, this churn cannot exhaust an identifier space or force the origin into matching replies.

This extension introduces no other security considerations beyond those described in {{moqt}}.


# IANA Considerations

This document requests the following registrations.

## MOQT Setup Options

This document requests a registration in the "MOQT Setup Options" registry ({{moqt}} Section 15.4), whose policy is Specification Required.
moq-transport defines no private-use range for Setup Options; extensions request a (provisional) codepoint.
A high, distinctive value is chosen to avoid the low ranges reserved by {{moqt}} and to minimize collisions with provisional registrations by other extensions; it also avoids the greasing pattern (`0x7f * N + 0x9D`).

| Value | Name | Reference |
|:------|:-----|:----------|
| 0xC0117 | SUBSCRIBE_DEMAND | This Document |

## MOQT Message Types

This document registers a control message type.
{{moqt}} does not yet establish an IANA registry for message types, so this is a provisional codepoint pending such a registry; the value is chosen to be high and distinctive to avoid the low ranges {{moqt}} assigns and to minimize collisions with provisional registrations by other extensions, and it avoids the greasing pattern (`0x7f * N + 0x9D`).
This is the same value as the SUBSCRIBE_DEMAND Setup Option above; Setup Options and message types are independent namespaces, so the shared value is unambiguous.

The Stream column has the meaning defined by {{moqt}} Section 10: "Request" indicates the message is carried on a bidirectional request stream. The message is not marked "First": it never opens a request stream.

| Value | Name | Stream | Reference |
|:------|:-----|:-------|:----------|
| 0xC0117 | SUBSCRIBE_DEMAND | Request | This Document |


--- back

# Acknowledgments
{:numbered="false"}

This document was drafted with the assistance of Claude, an AI assistant by Anthropic.
