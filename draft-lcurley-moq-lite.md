---
title: "Media over QUIC - Lite"
abbrev: "moql"
category: info

docname: draft-lcurley-moq-lite-latest
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

moq-lite is designed to fanout live content 1->N across the internet.
It leverages QUIC to prioritize important content, avoiding head-of-line blocking while respecting encoding dependencies.
While primarily designed for media, the transport is payload agnostic and can be proxied by relays/CDNs without knowledge of codecs, containers, or encryption keys.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Rationale
This draft is based on MoqTransport [moqt].
The concepts, motivations, and terminology are very similar and when in doubt, refer to existing MoqTransport literature.
A few things have been renamed (ex. object -> frame) to better align with media terminology.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

But it's been difficult to design such an experimental protocol via committee.
MoqTransport has become too complicated.

There are too many messages, optional modes, and half-baked features.
Too many hypotheses, too many potential use-cases, too many diametrically opposed opinions.
This is expected (and even desired) as compromise gives birth to a standard.

But I believe the standardization process is hindering practical experimentation.
The ideas behind MoQ can be proven now before being cemented as an RFC.
We should spend more time building an *actual* application and less time arguing about a hypothetical one.

moq-lite is the bare minimum needed for a real-time application aiming to replace WebRTC.
Every feature from MoqTransport that is not necessary (or has not been implemented yet) has been removed for simplicity.
This includes many great ideas (ex. group order) that may be added as they are needed.
This draft is the current state, not the end state.


# Concepts
moq-lite consists of:

- **Session**: An established QUIC connection between a client and server.
- **Broadcast**: A collection of Tracks from a single publisher.
- **Track**: An series of Groups, each of which can be delivered and decoded *out-of-order*.
- **Group**: An series of Frames, each of which must be delivered and decoded *in-order*.
- **Frame**: A sized payload of bytes representing a single moment in time.

The application determines how to split data into broadcast, tracks, groups, and frames.
The moq-lite layer provides fanout, prioritization, and caching even for latency sensitive applications.

## Session
A Session consists of a connection between a client and a server.
There is currently no P2P support within QUIC so it's out of scope for moq-lite.

A session is established after the necessary QUIC, WebTransport, and moq-lite handshakes have completed.
The moq-lite handshake is simple and consists of version and extension negotiation.

While moq-lite is a point-to-point protocol, it's intended to work end-to-end via relays.
Each client establishes a session with a CDN edge server, ideally the closest one.
Any broadcasts and subscriptions are transparently proxied by the CDN behind the scenes.

## Broadcast
A Broadcast is a collection of Tracks from a single publisher.
This corresponds to a MoqTransport's "track namespace".

A publisher may produce multiple broadcasts, each of which is advertised via an ANNOUNCE message.
The subscriber uses the ANNOUNCE_PLEASE message to discover available broadcasts.
These announcements are live and can change over time, allowing for dynamic origin discovery.

A broadcast consists of any number of Tracks.
The contents, relationships, and encoding of tracks are determined by the application.

## Track
A Track is a series of Groups identified by a unique name within a Broadcast.

A track consists of a single active Group at any moment, called the "latest group".
When a new Group is started, the previous Group is closed and may be dropped for any reason.
The duration before an incomplete group is dropped is determined by the application and the publisher/subscriber's latency target.

Every subscription is scoped to a single Track.
A subscription will always start at the latest Group and continues until either the publisher or subscriber cancels the subscription.

The subscriber and publisher both indicate their delivery preference:
- `Priority` indicates if Track A should be transmitted instead of Track B.
- `Ordered` indicates if the Groups within a Track should be transmitted in order.
- `Max Latency` indicates the maximum duration (based on instants) before a Group is abandoned.

The combination of these preferences enables the most important content to arrive during network degradation while still respecting encoding dependencies.

## Group
A Group is an ordered stream of Frames within a Track.

Each group consists of an append-only list of Frames.
A Group is served by a dedicated QUIC stream which is closed on completion, reset by the publisher, or cancelled by the subscriber.
This ensures that all Frames within a Group arrive reliably and in order.

In contrast, Groups may arrive out of order due to network congestion and prioritization.
The application MUST process or buffer groups out of order to avoid blocking on flow control

## Frame
A Frame is a payload of bytes within a Group.

A frame is used to represent a chunk of data with an upfront size and [instant](#instant).
The instant is relative to all other frames within the same Track, used for [expiration](#expiration) and metrics.
The application MAY use the instant as a replacement for a presentation timestamp, used for track playback and synchronization.

# Flow
This section outlines the flow of messages within a moq-lite session.
See the section for Messages section for the specific encoding.

## Connection
moq-lite runs on top of WebTransport.
WebTransport is a layer on top of QUIC and HTTP/3, required for web support.
The API is nearly identical to QUIC with the exception of stream IDs.

How the WebTransport connection is authenticated is out-of-scope for this draft.

## Termination
QUIC bidirectional streams have an independent send and receive direction.
Rather than deal with half-open states, moq-lite combines both sides.
If an endpoint closes the send direction of a stream, the peer MUST also close their send direction.

moq-lite contains many long-lived transactions, such as subscriptions and announcements.
These are terminated when the underlying QUIC stream is terminated.

To terminate a stream, an endpoint may:
- close the send direction (STREAM with FIN) to gracefully terminate (all messages are flushed).
- reset the send direction (RESET_STREAM) to immediately terminate.

After resetting the send direction, an endpoint MAY close the recv direction (STOP_SENDING).
However, it is ultimately the other peer's responsibility to close their send direction.

## Handshake
After a connection is established, the client opens a Session Stream and sends a SESSION_CLIENT message, to which the server replies with a SESSION_SERVER message.
The session is active until either endpoint closes or resets the Session Stream.

This session handshake is used to negotiate the moq-lite version and any extensions.
See the Extension section for more information.

# Streams
moq-lite uses a bidirectional stream for each transaction.
If the stream is closed, potentially with an error, the transaction is terminated.

## Bidirectional Streams
Bidirectional streams are used for control streams.
There's a 1-byte STREAM_TYPE at the beginning of each stream.

|---------|--------------|-------------|
|     ID  | Stream       | Creator     |
|--------:|:-------------|:------------|
|    0x0  | Session      | Client      |
| ------- | ------------ | ----------- |
|    0x1  | Announce     | Subscriber  |
| ------- | ------------- | ---------- |
|    0x2  | Subscribe    | Subscriber  |
| ------- | ------------- | ---------- |
|    0x20 | SessionCompat | Client     |
| ------- | ------------- | ----------- |

### Session
The Session stream is used to establish the moq-lite session, negotiating the version and any extensions.
This stream remains open for the duration of the session and its closure indicates the session is closed.

The client MUST open the Session Stream, write the Session Stream ID (0x0), and write a SESSION_CLIENT message.
If the server does not support any of the client's versions, it MUST close the stream with an error code and MAY close the connection.
Otherwise, the server replies with a SESSION_SERVER message to complete the handshake.

Afterwards, both endpoints MAY send SESSION_UPDATE messages.
This is currently used to notify the other endpoint of a significant change in the session bitrate.

This draft's version is combined with the constant `0xff0dad00`.
For example, moq-lite-draft-04 is identified as `0xff0dad04`.

### SessionCompat
The SessionCompat stream exists to support moq-transport draft 11-14.
This will be removed in a future version as moq-transport draft 15 uses ALPN instead.

The client writes a CLIENT_SETUP message on the SessionCompat stream and receives a SERVER_SETUP message in response.

Consult the MoqTransport ([moqt]) draft for more information about the encoding.
Notably, each message contains a u16 length prefix instead of a VarInt (moq-lite).

If a moq-lite version is negotiated, this stream becomes a normal Session stream.
If a moq-transport version is negotiated, this stream becomes the MoqTransport control stream.

### Announce
A subscriber can open a Announce Stream to discover broadcasts matching a prefix.

The subscriber creates the stream with a ANNOUNCE_PLEASE message.
The publisher replies with an ANNOUNCE_INIT message containing all currently active broadcasts that currently match the prefix, followed by ANNOUNCE messages for any changes.

The ANNOUNCE_INIT message contains an array of all currently active broadcast paths encoded as a suffix.
Each path in ANNOUNCE_INIT can be treated as if it were an ANNOUNCE message with status `active`.

After ANNOUNCE_INIT, the publisher sends ANNOUNCE messages for any changes also encoded as a suffix.
Each ANNOUNCE message contains one of the following statuses:

- `active`: a matching broadcast is available.
- `ended`: a previously `active` broadcast is no longer available.

Each broadcast starts as `ended` (unless included in ANNOUNCE_INIT) and MUST alternate between `active` and `ended`.
The subscriber MUST reset the stream if it receives a duplicate status, such as two `active` statuses in a row or an `ended` without `active`.
When the stream is closed, the subscriber MUST assume that all broadcasts are now `ended`.

Path prefix matching and equality is done on a byte-by-byte basis.
There MAY be multiple Announce Streams, potentially containing overlapping prefixes, that get their own ANNOUNCE_INIT and ANNOUNCE messages.

### Subscribe
A subscriber opens Subscribe Streams to request a Track.

The subscriber MUST start a Subscribe Stream with a SUBSCRIBE message followed by any number of SUBSCRIBE_UPDATE messages.
The publisher replies with any number of SUBSCRIBE_OK messages, or closes the stream.

The publisher SHOULD close the stream after the track has ended.
Either endpoint MAY reset/cancel the stream at any time.

# Delivery
The most important concept in moq-lite is how to deliver a subscription.
QUIC can only improve the user experience if data is delivered out-of-order during congestion.
This is the sole reason why data is divided into Broadcasts, Tracks, Groups, and Frames.

moq-lite consists of multiple groups being transmitted in parallel across seperate streams.
How these streams get transmitted over the network is very important, and yet has been distilled down into a few simple properties:

## Prioritization
The Publisher and Subscriber both exhchange `Priority` and `Ordered` values:
- `Priority` determines which Track should be transmitted next.
- `Ordered` determines when Group within the Track should be transmitted next.

A publisher SHOULD attempt to transmit streams based on these fields.
This depends on the QUIC library and it may not be possible to get fine-grained control.

### Priority
The `Subscriber Priority` is scoped to the connection.
The `Publisher Priority` SHOULD be used to resolve conflicts or ties.

A conflict can occur when a relay tries to serve multiple downstream subscriptions from a single upstream subscription.
Any upstream subscription SHOULD use the publisher priority, not some combination of different subscriber priorities.

Rather than try to explain everything, here's an example:

**Example:**
There are two people in a conference call, Ali and Bob.

We subscribe to both of their audio tracks with priority 2 and video tracks with priority 1.
This will cause equal priority for `Ali` and `Bob` while prioritizing audio.
```
ali/audio + bob/audio: subscriber_priority=2 publisher_priority=2
ali/video + bob/video: subscriber_priority=1 publisher_priority=1
```

If Bob starts actively speaking, they can bump their publisher priority via a SUBSCRIBE_OK message.
This would cause tracks be delivered in this order:
```
bob/audio: subscriber_priority=2 publisher_priority=3
ali/audio: subscriber_priority=2 publisher_priority=2
bob/video: subscriber_priority=1 publisher_priority=2
ali/video: subscriber_priority=1 publisher_priority=1
```

The subscriber priority takes presidence, so we could override it if we decided to full screen Ali's window:
```
ali/audio subscriber_priority=4 publisher_priority=2
ali/video subscriber_priority=3 publisher_priority=1
bob/audio subscriber_priority=2 publisher_priority=3
bob/audio subscriber_priority=1 publisher_priority=2
```

### Ordered
The `Subscriber Ordered` boolean signals if older (true) or newer (false) groups should be transmitted first within a Track.
The `Publisher Ordered` boolean MAY likewise be used to resolve conflicts.

An application SHOULD use `ordered` when it wants to provide a VOD-like experience, preferring to buffer old groups rather than skip them.
An application SHOULD NOT `ordered` when it wants to provide a live experience, preferring to skip old groups rather than buffer them.

Note that (expiration)[#expiration] is not affected by `ordered`.
An old group may still be cancelled/skipped if it's older than `max_latency` set by either peer.
An application MUST support gaps and out-of-order delivery even when `ordered` is true.


## Instant
Every Frame consists of an Instant (in milliseconds) scoped to the track.

This is used for [Expiration](#expiration) at the moq-lite layer.
The instant MAY be used for track synchronization at the application layer, however many containers/formats already contain their own timestamps.
A publisher SHOULD try to preserve the instant when the frame was first created/captured, proxying it.

Each frame within a group MUST have a monotonically increasing instant.
If the presentation timestamp goes backwards (ex. b-frames), a frame's Instant SHOULD be the maximum presentation timestamp of the group up until that point.
Duplicates are allowed, encoded as delta 0.

Unless specified by the application, a subscriber:
- SHOULD NOT assume that an instant is a wall clock time.
- SHOULD NOT assume that an instant starts at zero.

Clock synchronization is out of scope for this draft.

## Expiration
The Publisher and Subscriber both transmit a `Max Latency` value, indicating the maximum duration before a group is expired.

It is not crucial to aggressively expire groups thanks to [prioritization](#prioritization).
However, a lower priority group will still consume RAM, bandwidth, and potentially flow control.
It is RECOMMENDED that an application set conservative limits and only resort to expiration when data is absolutely no longer needed.

A subscriber SHOULD expire groups based on the `Subscriber Max Latency` in SUBSCRIBE/SUBSCRIBE_UPDATE.
A publisher SHOULD expire groups based on the `Publisher Max Latency` in SUBSCRIBE_OK.
An implementation MAY use the minimum of both when determining when to expire a group.

Each group consists of a maximum Frame Instant, the meaning of "processed" depending on the endpoint:
- A publisher uses a frame was queued, regardless of it it has been flushed to the QUIC layer.
- A subscriber uses when a frame was received, regardless of it it has been flushed to the application.

The entire track consists of a maximum Frame Instant using the same logic as above.
For each group, if the maximum Frame Instant is more than `Max Latency` behind the track's maximum Frame Instant, then the group is considered expired.
An expired group SHOULD BE reset at the QUIC level to avoid consuming flow control.


**Example:**
- Group 1: 1000 1500 (2000)
- Group 2: 2500 3000

Frame 2000 in this example has not been received yet by the subscriber.
If the `max_latency` was 1250, then the Subscriber SHOULD expire Group 1 but the Publisher SHOULD NOT yet.
The opposite would be true if Frame 3000 was still in transit:

**Example:**
- Group 1: 1000 1500 2000
- Group 2: 2500 (3000)

**NOTE**: Individual frames within a group cannot be cancelled.
Even if `max_latency` was 0, Group 2 would not be expired until a (higher) frame in Group 3 is queued/received.


An implementation MAY use the broadcast's maximum Frame Instant instead of the track's maximum Frame Instant.
This is useful to account for encoding delays, ex. when video takes 300ms longer to encode than audio.
However, it SHOULD NOT expire the highest sequence number group within each track, otherwise a track may become perpetually expired.

## Unidirectional Streams
Unidirectional streams are used for data transmission.

|--------|----------|-------------|
|     ID | Stream   | Creator     |
|-------:|:---------|-------------|
|    0x0 | Group    | Publisher   |
| ------ | -------- | ----------- |

### Group
A publisher creates Group Streams in response to a Subscribe Stream.

A Group Stream MUST start with a GROUP message and MAY be followed by any number of FRAME messages.
A Group MAY contain zero FRAME messages, potentially indicating a gap in the track.
A frame MAY contain an empty payload, potentially indicating a gap in the group.

Both the publisher and subscriber MAY reset the stream at any time.
This is not a fatal error and the session remains active.
The subscriber MAY cache the error and potentially retry later.



# Encoding
This section covers the encoding of each message.

## Message Length
Most messages are prefixed with a variable-length integer indicating the number of bytes in the message payload that follows.
This length field does not include the length of the varint length itself.

An implementation SHOULD close the connection with a PROTOCOL_VIOLATION if it receives a message with an unexpected length.
The version and extensions should be used to support new fields, not the message length.

## STREAM_TYPE
All streams start with a short header indicating the stream type.

~~~
STREAM_TYPE {
  Stream Type (i)
}
~~~

The stream ID depends on if it's a bidirectional or unidirectional stream, as indicated in the Streams section.
A receiver MUST close the session if it receives an unknown stream type.


## SESSION_CLIENT
The client initiates the session by sending a SESSION_CLIENT message.

~~~
SESSION_CLIENT Message {
  Message Length (i)
  Supported Versions Count (i)
  Supported Version (i)
  Extension Count (i)
  [
    Extension ID (i)
    Extension Payload (b)
  ]...
}
~~~


## SESSION_SERVER
The server responds with the selected version and any extensions.

~~~
SESSION_SERVER Message {
  Message Length (i)
  Selected Version (i)
  Extension Count (i)
  [
    Extension ID (i)
    Extension Payload (b)
  ]...
}
~~~

## SESSION_UPDATE

~~~
SESSION_UPDATE Message {
  Message Length (i)
  Session Bitrate (i)
}
~~~

**Session Bitrate**:
The estimated bitrate of the QUIC connection in bits per second.
This SHOULD be sourced directly from the QUIC congestion controller.
A value of 0 indicates that this information is not available.


## ANNOUNCE_PLEASE
A subscriber sends an ANNOUNCE_PLEASE message to indicate it wants to receive an ANNOUNCE message for any broadcasts with a path that starts with the requested prefix.

~~~
ANNOUNCE_PLEASE Message {
  Message Length (i)
  Broadcast Path Prefix (s),
}
~~~

**Broadcast Path Prefix**:
Indicate interest for any broadcasts with a path that starts with this prefix.

The publisher MUST respond with an ANNOUNCE_INIT message containing any matching and active broadcasts, followed by ANNOUNCE messages for any updates.
Implementations SHOULD consider reasonable limits on the number of matching broadcasts to prevent resource exhaustion.



## ANNOUNCE_INIT
A publisher sends an ANNOUNCE_INIT message immediately after receiving an ANNOUNCE_PLEASE to communicate all currently active broadcasts that match the requested prefix.
Only the suffixes are encoded on the wire, as the full path can be constructed by prepending the requested prefix.

This message is useful to avoid race conditions, as ANNOUNCE_INIT does not trickle in like ANNOUNCE messages.
For example, an API server that wants to list the current participants could issue an ANNOUNCE_PLEASE and immediately return the ANNOUNCE_INIT response.
Without ANNOUNCE_INIT, the API server would have use a timer to wait until ANNOUNCE to guess when all ANNOUNCE messages have been received.

~~~
ANNOUNCE_INIT Message {
  Message Length (i)
  Suffix Count (i),
  [
    Broadcast Path Suffix (s),
  ]...
}
~~~

**Suffix Count**:
The number of active broadcast path suffixes that follow.
This can be 0.
A publisher MUST NOT include duplicate suffixes in a single ANNOUNCE_INIT message.

**Broadcast Path Suffix**:
Each suffix is combined with the broadcast path prefix from ANNOUNCE_PLEASE to form the full broadcast path.
This includes all currently active broadcasts matching the prefix.



## ANNOUNCE
A publisher sends an ANNOUNCE message to advertise a change in broadcast availability.
Only the suffix is encoded on the wire, as the full path can be constructed by prepending the requested prefix.

The status is relative to the ANNOUNCE_INIT and all prior ANNOUNCE messages combined.
A client MUST ONLY alternate between status values (from active to ended or vice versa).

~~~
ANNOUNCE Message {
  Message Length (i)
  Announce Status (i),
  Broadcast Path Suffix (s),
}
~~~

**Announce Status**:
A flag indicating the announce status.

- `ended` (0): A path is no longer available.
- `active` (1): A path is now available.

**Broadcast Path Suffix**:
This is combined with the broadcast path prefix to form the full broadcast path.


## SUBSCRIBE
SUBSCRIBE is sent by a subscriber to start a subscription.

~~~
SUBSCRIBE Message {
  Message Length (i)
  Subscribe ID (i)
  Broadcast Path (s)
  Track Name (s)
  Subscriber Priority (8)
  Subscriber Ordered (1)
  Subscriber Max Latency (i)
}
~~~

**Subscribe ID**:
A unique identifier chosen by the subscriber.
A Subscribe ID MUST NOT be reused within the same session, even if the prior subscription has been closed.

**Subscriber Priority**:
The priority of the subscription within the session, represented as a u8.
The publisher SHOULD transmit *higher* values first during congestion.
See the [Prioritization](#prioritization) section for more information.

**Subscriber Ordered**:
A boolean representing whether groups are transmitted in ascending (true) or descending (false) order.
The publisher SHOULD transmit *older* groups first during congestion if true.
See the [Prioritization](#prioritization) section for more information.

**Subscriber Max Latency**:
This value is encoded in milliseconds and represents the maximum age of a group.
The publisher SHOULD reset old group streams when the last processed frame is at least this much older than the newest frame.
See the [Expiration](#expiration) section for more information.


## SUBSCRIBE_UPDATE
A subscriber can modify a subscription with a SUBSCRIBE_UPDATE message.
A subscriber MAY send multiple SUBSCRIBE_UPDATE messages to update the subscription.

~~~
SUBSCRIBE_UPDATE Message {
  Message Length (i)
  Subscriber Priority (i)
  Subscriber Ordered (1)
  Subscriber Max Latency (i)
}
~~~

See [SUBSCRIBE](#subscribe) for information about each field.


## SUBSCRIBE_OK
A SUBSCRIBE_OK message is sent in response to a SUBSCRIBE.
The publisher MAY send multiple SUBSCRIBE_OK messages to update the subscription.

~~~
SUBSCRIBE_OK Message {
  Message Length (i)
  Publisher Priority (8)
  Publisher Ordered (1)
  Publisher Max Latency (i)
}
~~~

See [SUBSCRIBE](#subscribe) for information about each field.

## GROUP
The GROUP message contains information about a Group, as well as a reference to the subscription being served.

~~~
GROUP Message {
  Message Length (i)
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

**Subscribe ID**:
The corresponding Subscribe ID.
This ID is used to distinguish between multiple subscriptions for the same track.

**Group Sequence**:
The sequence number of the group.
This SHOULD increase by 1 for each new group.
A subscriber MUST handle gaps, potentially caused by congestion.


## FRAME
The FRAME message is a payload at a specific point of time.

~~~
FRAME Message {
  Message Length (i)
  Instant Delta (i)
  Payload (b)
}
~~~

**Instant Delta**:
The instant when the frame was created, in milliseconds, scoped to the track.
This is encoded as a delta from the previous frame in the same group.
An application SHOULD use the capture/presentation time to account for any encoding delays.
See the [Instant](#instant) section for more information.

**Payload**:
An application specific payload.
A generic library or relay MUST NOT inspect or modify the contents unless otherwise negotiated.


# Appendix A: Changelog

## moq-lite-03
- Added `Subscriber Max Latency` and `Subscriber Ordered` to SUBSCRIBE and SUBSCRIBE_UPDATE.
- Added `Publisher Priority`, `Publisher Max Latency`, and `Publisher Ordered` to SUBSCRIBE_OK.
- Added `Instant Delta` to FRAME.
- SUBSCRIBE_OK may be sent multiple times.

## moq-lite-02
- Added SessionCompat stream.
- Editorial stuff.

## moq-lite-01
- Added ANNOUNCE_INIT.
- Added Message Length (i) to all messages.

# Appendix B: Upstream Differences
A quick comparison of moq-lite and moq-transport-14:

- Streams instead of request IDs.
- Pull only: No unsolicited publishing.
- Uses HTTP for VOD instead of FETCH.
- Extensions instead of parameters.
- Names use utf-8 strings instead of byte arrays.
- Track Namespace is a string, not an array of any array of bytes.
- Subscriptions start at the latest group, not the latest object.
- No subgroups
- No group/object ID gaps
- No object properties
- No datagrams
- No paused subscriptions (forward=0)

## Deleted Messages
- GOAWAY
- MAX_SUBSCRIBE_ID
- REQUESTS_BLOCKED
- SUBSCRIBE_ERROR
- UNSUBSCRIBE
- PUBLISH_DONE
- PUBLISH
- PUBLISH_OK
- PUBLISH_ERROR
- FETCH
- FETCH_OK
- FETCH_ERROR
- FETCH_CANCEL
- FETCH_HEADER
- TRACK_STATUS
- TRACK_STATUS_OK
- TRACK_STATUS_ERROR
- PUBLISH_NAMESPACE
- PUBLISH_NAMESPACE_OK
- PUBLISH_NAMESPACE_ERROR
- PUBLISH_NAMESPACE_CANCEL
- SUBSCRIBE_NAMESPACE_OK
- SUBSCRIBE_NAMESPACE_ERROR
- UNSUBSCRIBE_NAMESPACE
- OBJECT_DATAGRAM

## Renamed Messages
- SUBSCRIBE_NAMESPACE -> ANNOUNCE_PLEASE
- SUBGROUP_HEADER -> GROUP

## Deleted Fields
Some of these fields occur in multiple messages.

- Request ID
- Track Alias
- Group Order
- Filter Type
- StartGroup
- StartObject
- EndGroup
- Expires
- ContentExists
- Largest Group ID
- Largest Object ID
- Parameters
- Subgroup ID
- Object ID
- Object Status
- Extension Headers


# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
