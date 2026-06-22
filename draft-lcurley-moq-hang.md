---
title: "Media over QUIC - Hang"
abbrev: "hang"
category: info

docname: draft-lcurley-moq-hang-latest
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
  moql: I-D.lcurley-moq-lite
  moqt: I-D.ietf-moq-transport
  webcodecs: WebCodecs

informative:

--- abstract

Hang is a real-time conferencing protocol built on top of moq-lite.
A room consists of multiple participants who publish media tracks.
All updates are live, such as a change in participants or media tracks.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Terminology
Hang is built on top of moq-lite [moql] and uses much of the same terminology.
A quick recap:

- **Broadcast**: A collection of Tracks from a single publisher.
- **Track**: An series of Groups, each of which can be delivered and decoded *out-of-order*.
- **Group**: An series of Frames, each of which must be delivered and decoded *in-order*.
- **Frame**: A sized payload of bytes representing a single moment in time.

Hang introduces additional terminology:

- **Room**: A collection of participants, publishing under a common prefix.
- **Participant**: A moq-lite broadcaster that may produce any number of media tracks.
- **Catalog**: A JSON document that describes each available media track, supporting live updates.
- **Container**: A tiny header in front of each media payload containing the timestamp.
- **Timeline**: An optional index mapping each media Group to the presentation timestamp of its first frame, used for seeking.


# Discovery
The first requirement for a real-time conferencing application is to discover other participants in the same room.
Hang does this using moq-lite's ANNOUNCE capabilities.

A room consists of a path.
Any participants within the room MUST publish a broadcast with the room path as a prefix which SHOULD end with the `.hang` suffix.

For example:

~~~
/room123/alice.hang
/room123/bob.hang
/room456/zoe.hang
~~~

A participant issues an ANNOUNCE_PLEASE message to discover any other participants in the same room.
The server (relay) will then respond with an ANNOUNCE message for any matching broadcasts, including their own.

For example:

~~~
ANNOUNCE_PLEASE prefix=/room/
ANNOUNCE suffix=alice.hang active=true
ANNOUNCE suffix=bob.hang   active=true
~~~

If a publisher no longer wants to participant, or is disconnected somehow, their presence will be unannounced.
Publishers and subscribers SHOULD terminate any subscriptions once a participant is unannounced.

~~~
ANNOUNCE suffix=alice.hang active=false
~~~

# Catalog
The catalog describes the available media tracks for a single participant.
It's a JSON document that extends the the W3C WebCodecs specification.

The catalog is published as a `catalog.json` track within the broadcast so it can be updated live as the participant's media tracks change.
A participant MAY forgo publishing a catalog if it does not wish to publish any media tracks now and in the future.

The catalog track consists of multiple groups, one for each update.
Each group contains a single frame with UTF-8 JSON.

A publisher MUST NOT write multiple frames to a group until a future specification includes a delta-encoding mechanism (via JSON Patch most likely).

## Root
The root of the catalog is a JSON document with the following schema:

~~~
type Catalog = {
	"audio": AudioSchema | undefined,
	"video": VideoSchema | undefined,
	// ... any custom fields ...
}
~~~

Additional fields MAY be added based on the application.
The catalog SHOULD be mostly static, delegating any dynamic content to other tracks.

For example, a `"chat"` section should include the name of a chat track, not individual chat messages.
This way catalog updates are rare and a client MAY choose to not subscribe.

This specification currently only defines audio and video tracks.

## Video
A video track contains the necessary information to decode a video stream.


~~~
type VideoSchema = {
	"renditions": Map<TrackName, VideoDecoderConfig>,
	"timeline": TrackName | undefined,
	"priority": u8,
	"display": {
		"width": number,
		"height": number,
	} | undefined,
	"rotation": number | undefined,
	"flip": boolean | undefined,
}
~~~

The `renditions` field contains a map of track names to video decoder configurations.
See the [WebCodecs specification](https://www.w3.org/TR/webcodecs/#video-decoder-config) for specifics and registered codecs.
Any Uint8Array fields are hex-encoded as a string.

The optional `timeline` field names a [Timeline](#timeline) track that indexes this media for seeking.

For example:

~~~
{
	"renditions": {
		"720p": {
			"codec": "avc1.64001f",
			"codedWidth": 1280,
			"codedHeight": 720,
			"bitrate": 6000000,
			"framerate": 30.0
		},
		"480p": {
			"codec": "avc1.64001e",
			"codedWidth": 848,
			"codedHeight": 480,
			"bitrate": 2000000,
			"framerate": 30.0
		}
	},
	"timeline": "video/timeline",
	"priority": 2,
	"display": {
		"width": 1280,
		"height": 720
	},
	"rotation": 0,
	"flip": false,
}
~~~


## Audio
An audio track contains the necessary information to decode an audio stream.

~~~
type AudioSchema = {
	"renditions": Map<TrackName, AudioDecoderConfig>,
	"timeline": TrackName | undefined,
	"priority": u8,
}
~~~

The `renditions` field contains a map of track names to audio decoder configurations.
See the [WebCodecs specification](https://www.w3.org/TR/webcodecs/#audio-decoder-config) for specifics and registered codecs.
Any Uint8Array fields are hex-encoded as a string.

The optional `timeline` field names a [Timeline](#timeline) track that indexes this media for seeking.

For example:

~~~
{
	"renditions": {
		"stereo": {
			"codec": "opus",
			"sampleRate": 48000,
			"numberOfChannels": 2,
			"bitrate": 128000
		},
		"mono": {
			"codec": "opus",
			"sampleRate": 48000,
			"numberOfChannels": 1,
			"bitrate": 64000
		}
	},
	"timeline": "audio/timeline",
	"priority": 1,
}
~~~

# Container
Audio and video tracks use a lightweight container to encapsulate the media payload.

Each moq-lite group MUST start with a keyframe.
If codec does not support delta frames (ex. audio), then a group MAY consist of multiple keyframes.
Otherwise, a group MUST consist of a single keyframe followed by zero or more delta frames.

Each frame starts with a timestamp, a QUIC variable-length integer (62-bit max) encoded in microseconds.
The remainder of the payload is codec specific; see the WebCodecs specification for specifics.

For example, h.264 with no `description` field would be annex.b encoded, while h.264 with a `description` field would be AVCC encoded.


# Timeline
The timeline is an optional track that indexes a media track's Groups by presentation timestamp.
It lets a subscriber seek — resolving a presentation timestamp to a Group sequence — without first downloading and parsing the start of every Group.
It carries no media payload and is typically given a lower priority than the media tracks it indexes.

A media schema references its timeline track by name via the `timeline` field (see [Catalog](#catalog)).
A timeline reflects the Group and timestamp structure of one media track.
The renditions of a single media already share that structure — matching Group sequence numbers and per-group timestamps so a subscriber can switch between them — so one timeline serves all of a media's renditions.
Audio and video do not share that structure and use separate timelines.

The timeline track is a single Group that grows over the lifetime of the broadcast.
Each frame appends one entry, in Group order, mapping a Group to the presentation timestamp of that Group's first frame.
A subscriber builds the full index by reading the Group from its first frame; because moq-lite always retains the latest Group, a subscriber that joins late still receives every prior entry.
A publisher SHOULD keep the whole timeline in this single Group rather than starting a new one: moq-lite guarantees retention only of the latest Group, so splitting the index across Groups risks the earlier entries being dropped before a late subscriber can read them.

To seek to a presentation timestamp, a subscriber selects the entry with the greatest timestamp not exceeding the target and subscribes to (or fetches) that Group, which begins with a keyframe (see [Container](#container)).

## Encoding
The timeline reuses the moq-lite `FRAME` to carry the presentation timestamp, leaving a single value in the payload:

~~~
Timeline Frame {
  Group Delta (i),
}
~~~

The presentation timestamp is expressed in the track's `Publisher Timescale` ([moql]).
This is application-defined and SHOULD match the timescale of the media track being indexed so the values are directly comparable when seeking.

**Group Delta**:
The increase in Group sequence from the previous entry, encoded as an unsigned variable-length integer.
The first entry in the Group encodes the absolute Group sequence (a delta from 0).
This is normally `1`, since a publisher assigns Group sequences sequentially, but a larger value lets a timeline index Groups sparsely (for example, only every Nth Group).

**Presentation Timestamp**:
The presentation timestamp of the Group's first frame is not stored in the payload; it is carried by the moq-lite frame's `Timestamp Delta` field ([moql]), a zigzag-encoded signed delta from the previous entry in the track's timescale.
The first entry in the Group therefore carries the absolute timestamp, following the moq-lite frame rules.
A signed delta also accommodates the occasional backward step caused by frame reordering (e.g. an open GOP).

For example, a video track using a microsecond timescale with a constant two-second GOP starting at Group 100 produces these entries (the timestamp is carried by the moq-lite frame, the group delta by the payload):

~~~
entry 0: timestamp=0          group=100   # absolute anchor
entry 1: timestamp=+2000000   group=+1
entry 2: timestamp=+2000000   group=+1
~~~

Each entry occupies only a few bytes, and a regular cadence repeats identical deltas, so the index stays small even for long broadcasts.


# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
