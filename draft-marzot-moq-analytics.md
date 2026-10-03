---
title: "Metrics and Analytics Delivery for Media over QUIC Transport"
abbrev: "MoQT Analytics"
category: std

docname: draft-marzot-moq-analytics-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - metrics
 - analytics
 - observability
 - QoE
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "gmarzot/draft-marzot-moq-analytics"
  latest: "https://gmarzot.github.io/draft-marzot-moq-analytics/draft-marzot-moq-analytics.html"

author:
 -
    fullname: Giovanni Marzot
    organization: Oracle
    email: giovanni.marzot@oracle.com

normative:
  MOQT: I-D.ietf-moq-transport
  MSF: I-D.ietf-moq-msf
  JSON: RFC8259
  CBOR: RFC8949
  CDDL: RFC8610
  RFC9002:

informative:
  LOC: I-D.ietf-moq-loc
  CMSF: I-D.ietf-moq-cmsf
  MOQFEEDBACK: I-D.liu-moq-feedback
  MOQMETRICS: I-D.jennings-moq-metrics
  MOQ-QLOG: I-D.pardue-moq-qlog-moq-events
  SECURE-OBJECTS: I-D.ietf-moq-secure-objects
  RFC9000:
  WEBRTC-STATS:
    title: "Identifiers for WebRTC's Statistics API"
    target: https://www.w3.org/TR/webrtc-stats/
    author:
      org: W3C
    date: false
  CMCD:
    title: "Web Application Video Ecosystem - Common Media Client Data"
    target: https://github.com/cta-wave/common-media-client-data
    author:
      org: Consumer Technology Association
    seriesinfo:
      CTA: "5004"
    date: false
  CMSD:
    title: "Web Application Video Ecosystem - Common Media Server Data"
    target: https://github.com/cta-wave/common-media-server-data
    author:
      org: Consumer Technology Association
    seriesinfo:
      CTA: "5006"
    date: false
  OPENMETRICS:
    title: "OpenMetrics"
    target: https://github.com/prometheus/OpenMetrics/blob/main/specification/OpenMetrics.md
    author:
      org: Prometheus Authors
    date: false
  OTEL:
    title: "OpenTelemetry Metrics Data Model"
    target: https://opentelemetry.io/docs/specs/otel/metrics/data-model/
    author:
      org: OpenTelemetry Authors
    date: false
  PROTOBUF:
    title: "Protocol Buffers"
    target: https://protobuf.dev/
    author:
      org: Google
    date: false

--- abstract

Media over QUIC Transport (MoQT) deployments need metrics that can be
interpreted consistently across publishers, subscribers, relays, and
collectors, and a way to deliver those metrics without a separate monitoring
transport.

This document defines a set of MoQT metric sets, modeled on WebRTC statistics:
typed groups of optional, camelCase members describing sessions, transports,
tracks, subscriptions, media pipelines, playback, and relays. It defines a
Metric Report that carries metric sets, specified in CDDL with JSON and CBOR
encodings. It also defines how Metric Reports are delivered as objects on MoQT
analytics tracks, either to an endpoint directed by an MSF or CMSF catalog or
to a collector configured by the reporter.

--- middle

# Introduction

MoQT {{MOQT}} is a media-agnostic publish/subscribe protocol running over QUIC
and WebTransport. It exposes protocol events and delivery coordinates, such as
sessions, requests, namespaces, tracks, groups, and objects, that are useful for
operational measurement. A deployment may also have a media pipeline that
exposes codec, frame, buffer, and playback statistics.

These measurements are currently difficult to combine. One implementation may
expose subscription counters, another may expose QUIC recovery state, and a
media endpoint may expose WebRTC statistics {{WEBRTC-STATS}}. The same concept
may have different names, units, and aggregation rules. A relay operator
therefore cannot reliably compare a publisher's observations with a subscriber's
observations or correlate them with relay and transport events.

MSF {{MSF}} allows a catalog to declare publish tracks to which a subscriber
sends metrics, and {{MOQMETRICS}} defines a generic metrics payload that carries
one named gauge or counter per object. Neither defines what MoQT-specific
metrics mean. This document fills that gap and defines a delivery format suited
to it.

## Overview

This document defines three things:

Metric sets ({{sets}}):
: Typed groups of related metrics, each describing one measured entity such as
  a transport, an inbound track, or a playback session. As with WebRTC
  statistics, every member other than the identifying members is optional, so
  that a reporter includes only what it can measure.

Metric Reports ({{report}}):
: A container that carries a batch of metric sets from one reporter, specified
  in CDDL {{CDDL}} with a JSON {{JSON}} encoding and a CBOR {{CBOR}} encoding.

Delivery ({{delivery}}):
: A mapping of Metric Reports onto the objects of a MoQT analytics track,
  integration with the MSF and CMSF catalog, and delivery to a collector
  configured by the reporter.

The metric sets are useful independently of the delivery mechanism. An
implementation can also export them to OpenMetrics {{OPENMETRICS}} or
OpenTelemetry {{OTEL}} using the naming rule in {{export}}.

## Non-Goals

This document does not define a universal playback model, a replacement for
QUIC congestion-control feedback, or per-object delivery feedback such as that
defined in {{MOQFEEDBACK}}. It does not require a node to report every metric
set or member.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms defined in {{MOQT}}, including Session, Track,
Track Namespace, Group, Object, Publisher, Subscriber, and Relay. In addition:

Reporter:
: A publisher, subscriber, or relay that produces metric sets.

Collector:
: A component that receives, stores, aggregates, or presents metric sets.

Metric Set:
: A typed collection of members describing one measured entity at one point in
  time.

Member:
: A named value within a metric set.

Metric Report:
: A container carrying one or more metric sets from a single reporter.

Analytics Track:
: A MoQT track whose objects carry Metric Reports.

Member and field names in this document use lowerCamelCase, consistent with
{{WEBRTC-STATS}} and the MSF catalog. Metric set type names use lowercase words
separated by hyphens.


# Metric Model {#model}

## Measurement Layers

Members are measured at one of the following layers:

* MoQT layer: requests, subscriptions, objects, groups, and streams;
* QUIC layer: round-trip time, loss, congestion control, flow control, and
  datagrams;
* media layer: encoding, decoding, frames, and codecs; and
* playback layer: buffering, latency, stalls, and rendering.

Each metric set type in {{sets}} belongs to one layer. A reporter MUST NOT
report a value measured at one layer in a member defined at another layer. For
example, a subscriber that has no playback signal cannot report a playback
stall based on transport observations alone.

## Value Types {#types}

Each member has one of the following value types:

Counter:
: A cumulative, non-decreasing value over the lifetime of the metric set. Counts
  are unsigned integers. Cumulative times are numbers in seconds.

Gauge:
: A value sampled at the metric set's timestamp that may increase or decrease.

Histogram:
: A distribution of observations made over the lifetime of the metric set; see
  {{histograms}}.

Attribute:
: A descriptive value, such as a codec string or a state, that is not a
  measurement.

Durations are expressed in seconds, bitrates in bits per second, and sizes in
bytes. Timestamps are expressed in milliseconds since the Unix epoch,
consistent with the MSF catalog.

## Optionality {#optionality}

Every member of a metric set other than `type`, `id`, and `timestamp` is
OPTIONAL. A reporter MUST omit a member that it cannot measure or derive as
defined, rather than reporting zero or a placeholder value. A collector MUST
interpret an absent member as "not reported", not as zero.

## Lifetime and Resets

A metric set describes an entity, such as a transport or a track, for as long
as that entity exists at the reporter. Counters and histograms accumulate from
the time the reporter first creates the metric set. The `id` of a metric set
MUST remain stable for its lifetime, and a reporter MUST use a new `id` whenever
the counters of a metric set are reset, for example after a process restart. A
collector can therefore treat a change of `id` as a counter reset.

## Identifiers and Cardinality {#cardinality}

Metric sets refer to one another by `id`, for example an `inbound-track` set
names its session in `sessionId`. These identifiers are scoped to the reporter
and are not meaningful to other nodes.

The principal operational scope is normally the track. Group and object
identifiers are useful for local correlation and debugging, but are not
suitable as dimensions of a time series. Reporters SHOULD aggregate group- and
object-level observations into counters or histograms, as the metric sets in
this document do.

Track namespaces and track names can reveal content and viewing behavior, and
can have high cardinality across a deployment. A reporter MAY omit `namespace`
and `trackName` and identify a track only by the `id` of its metric set when the
collector already knows the association, or when policy does not permit
reporting the names.


# Metric Sets {#sets}

Each metric set has the following members:

`type`:
: Attribute, required. The metric set type, as defined in this section or as an
  extension type ({{extensions}}).

`id`:
: Attribute, required. An identifier for the metric set, unique within the
  reporter for the lifetime of the set.

`timestamp`:
: Required. The time at which the members of the set were sampled, in
  milliseconds since the Unix epoch.

The following metric set types are defined:

| Type | Layer | Describes |
|---|---|---|
| `session` | MoQT | A MoQT session |
| `requests` | MoQT | Requests of one kind |
| `transport` | QUIC | The QUIC connection of a session |
| `outbound-track` | MoQT | A track as sent by the reporter |
| `inbound-track` | MoQT | A track as received by the reporter |
| `subscription` | MoQT | One subscription |
| `media-source` | media | An encoder output for a track |
| `inbound-media` | media | A decoder input for a track |
| `playback` | playback | Playback of a set of tracks |
| `relay` | MoQT | A relay node |
{: title="Metric Set Types"}

## session

`transportId`:
: Attribute. The `id` of the `transport` set for this session.

`version`:
: Attribute. The negotiated MoQT version.

`binding`:
: Attribute. `quic` or `webtransport`.

`state`:
: Attribute. One of `connecting`, `established`, `draining`, or `closed`.

`setupTime`:
: Gauge, seconds. Time from the start of the connection to completion of the
  MoQT session setup.

`goawaysReceived`:
: Counter. GOAWAY messages received.

## requests

A `requests` set summarizes requests of one kind on one session, or across the
reporter when `sessionId` is absent.

`kind`:
: Attribute, required. One of `subscribe`, `fetch`, `publish`,
  `publish-namespace`, `subscribe-namespace`, `track-status`, or a request kind
  defined by an extension to {{MOQT}}.

`sessionId`:
: Attribute. The `id` of the `session` set.

`sent`:
: Counter. Requests sent.

`received`:
: Counter. Requests received.

`succeeded`:
: Counter. Requests that completed successfully.

`failed`:
: Counter. Requests that ended in an error.

`active`:
: Gauge. Requests currently in progress.

`responseTime`:
: Histogram, seconds. Time from sending a request to receiving its response.

`errorCodes`:
: Counter. Failed requests by error code, as a list of code and count pairs.

## transport

A `transport` set describes the QUIC connection {{RFC9000}} underlying a
session. The RTT members have the meanings defined in {{RFC9002, Section 5}}.

`sessionId`:
: Attribute. The `id` of the `session` set.

`bytesSent`, `bytesReceived`:
: Counter, bytes. UDP payload bytes sent and received.

`packetsSent`, `packetsReceived`:
: Counter. QUIC packets sent and received.

`packetsLost`:
: Counter. QUIC packets declared lost.

`latestRtt`, `minRtt`, `smoothedRtt`, `rttVar`:
: Gauge, seconds. The latest_rtt, min_rtt, smoothed_rtt, and rttvar values.

`cwnd`:
: Gauge, bytes. The congestion window.

`bytesInFlight`:
: Gauge, bytes. Bytes sent but not yet acknowledged or declared lost.

`availableBitrate`:
: Gauge, bits per second. The sending rate the congestion controller estimates
  is available.

`cwndBlockedTime`:
: Counter, seconds. Time during which sending was blocked by congestion control.

`flowBlockedTime`:
: Counter, seconds. Time during which sending was blocked by flow control.

`streamsOpened`:
: Counter. Streams opened.

`streamsActive`:
: Gauge. Streams currently open.

`datagramsSent`, `datagramsReceived`:
: Counter. QUIC datagrams sent and received.

`datagramsLost`:
: Counter. QUIC datagrams declared lost.

## outbound-track

An `outbound-track` set describes a track that the reporter sends on one
session. Byte counts in track sets count object payload bytes.

`namespace`, `trackName`:
: Attribute. The track namespace, as a list of strings, and the track name.

`sessionId`:
: Attribute. The `id` of the `session` set.

`deliveryMode`:
: Attribute. `subgroup`, `datagram`, or `mixed`.

`priority`:
: Gauge. The publisher priority of the track.

`objectsSent`:
: Counter. Objects sent.

`bytesSent`:
: Counter, bytes. Object payload bytes sent.

`groupsSent`:
: Counter. Groups for which at least one object was sent.

`objectsDropped`:
: Counter. Objects that the reporter chose not to send, for example due to a
  delivery timeout or local policy.

`streamsReset`:
: Counter. Streams for this track reset by the reporter.

`subscriptions`:
: Gauge. Active subscriptions served by this track.

`largestLocation`:
: Gauge. The largest group and object sent.

## inbound-track

An `inbound-track` set describes a track that the reporter receives on one
session.

`namespace`, `trackName`, `sessionId`, `deliveryMode`:
: As for `outbound-track`.

`objectsReceived`:
: Counter. Objects received.

`bytesReceived`:
: Counter, bytes. Object payload bytes received.

`groupsReceived`:
: Counter. Groups for which at least one object was received.

`groupsCompleted`:
: Counter. Groups received in full, as determined by the end of each group.

`objectsLost`:
: Counter. Objects determined not to have been received. A reporter MUST NOT
  count an object as lost only because it has not yet arrived.

`objectsLate`:
: Counter. Objects received after an application-defined deadline.

`duplicates`:
: Counter. Objects received more than once.

`streamsReset`:
: Counter. Streams for this track reset by the sender.

`largestLocation`:
: Gauge. The largest group and object received.

`interarrival`:
: Histogram, seconds. Time between consecutive received objects.

`latency`:
: Histogram, seconds. Time from object creation to receipt. This member
  requires a creation timestamp and a clock synchronized with the sender.

`groupDuration`:
: Histogram, seconds. Time from the first to the last object received in a
  group.

## subscription

`namespace`, `trackName`, `sessionId`:
: As for `outbound-track`.

`trackId`:
: Attribute. The `id` of the `inbound-track` or `outbound-track` set.

`requestId`:
: Attribute. The MoQT request ID of the subscription.

`state`:
: Attribute. One of `pending`, `active`, `done`, or `error`.

`forward`:
: Attribute. The forward state of the subscription.

`priority`:
: Gauge. The subscriber priority.

`groupOrder`:
: Attribute. `ascending` or `descending`.

`responseTime`:
: Gauge, seconds. Time from sending the subscription request to receiving its
  response.

`repairRequests`:
: Counter. FETCH requests issued to recover objects missing from this
  subscription.

## media-source

A `media-source` set describes the output of an encoder at a publisher. Where a
member shares a name with a member of the WebRTC `outbound-rtp` or
`media-source` statistics {{WEBRTC-STATS}}, it has the same definition.

`trackId`:
: Attribute. The `id` of the `outbound-track` set carrying this media.

`kind`:
: Attribute. `audio`, `video`, or another media kind.

`codec`:
: Attribute. The codec string, as used in the MSF catalog.

`bitrate`:
: Gauge, bits per second. The encoded bitrate.

`framesEncoded`:
: Counter. Frames encoded.

`keyFramesEncoded`:
: Counter. Key frames encoded.

`framesPerSecond`:
: Gauge. Frames encoded per second.

`keyFrameInterval`:
: Gauge, seconds. Time between key frames.

`width`, `height`:
: Gauge, pixels. The encoded frame dimensions.

`totalEncodeTime`:
: Counter, seconds. Time spent encoding.

## inbound-media

An `inbound-media` set describes the input to a decoder at a subscriber. Where a
member shares a name with a member of the WebRTC `inbound-rtp` statistics
{{WEBRTC-STATS}}, it has the same definition.

`trackId`:
: Attribute. The `id` of the `inbound-track` set carrying this media.

`kind`, `codec`:
: As for `media-source`.

`bitrate`:
: Gauge, bits per second. The received media bitrate.

`framesReceived`:
: Counter. Frames received.

`framesDecoded`:
: Counter. Frames decoded.

`keyFramesDecoded`:
: Counter. Key frames decoded.

`framesDropped`:
: Counter. Frames dropped before or after decoding.

`framesPerSecond`:
: Gauge. Frames decoded per second.

`totalDecodeTime`:
: Counter, seconds. Time spent decoding.

`jitter`:
: Gauge, seconds. Variation in media arrival time, as defined by the media
  pipeline. This is distinct from the `interarrival` histogram of the
  `inbound-track` set.

`width`, `height`:
: Gauge, pixels. The decoded frame dimensions.

`concealedSamples`:
: Counter. Audio samples concealed.

## playback

A `playback` set describes the playback of one or more tracks at a
subscriber. All of its members require a playback model.

`trackIds`:
: Attribute. The `id` values of the `inbound-media` or `inbound-track` sets
  being played.

`renderGroup`:
: Attribute. The MSF render group being played.

`state`:
: Attribute. One of `startup`, `playing`, `paused`, `stalled`, or `ended`.

`startupTime`:
: Gauge, seconds. Time from the start of playback to the first rendered media.

`bufferLevel`:
: Gauge, seconds. Media buffered ahead of the playback position.

`targetLatency`:
: Gauge, seconds. The latency the player is attempting to maintain.

`liveLatency`:
: Gauge, seconds. Time between the creation of the media being rendered and its
  rendering. This member requires a clock synchronized with the publisher.

`deadline`:
: Gauge, seconds. Time remaining until the next object must be available to
  avoid a stall.

`stallCount`:
: Counter. Playback stalls after startup.

`totalStallTime`:
: Counter, seconds. Time spent stalled after startup.

`nonRendered`:
: Counter. Objects received but not rendered, for example because they arrived
  after their playback time.

`switches`:
: Counter. Switches between tracks in an alternate group.

## relay

`sessions`:
: Gauge. Active sessions.

`subscriptions`:
: Gauge. Active subscriptions.

`fetches`:
: Gauge. Active FETCH requests.

`cacheHits`, `cacheMisses`:
: Counter. Object lookups satisfied, and not satisfied, by the relay cache.

`cacheBytes`:
: Gauge, bytes. Object payload bytes held in the cache.

`processingTime`:
: Histogram, seconds. Time from receiving a request to forwarding it or
  responding.

`heldTime`:
: Histogram, seconds. Time objects were held in a relay queue before being sent.

`availabilityTime`:
: Histogram, seconds. Time from receiving an object to it being available to
  subscribers.

`duress`:
: Gauge. A deployment-defined load indicator between 0 and 1.

## Histograms {#histograms}

A histogram member is a map with the following fields:

* `count`: the number of observations;
* `sum`: the sum of the observations;
* `min` and `max`: OPTIONAL minimum and maximum observations;
* `bounds`: an OPTIONAL list of ascending bucket upper bounds; and
* `counts`: the number of observations in each bucket, with one more entry than
  `bounds`. The last entry counts observations greater than the last bound.

This is the explicit bucket histogram of {{OTEL}}. Bucket bounds are chosen by
the reporter unless a catalog or profile specifies them.

## Extensions {#extensions}

A reporter MAY include members and metric set types not defined in this
document. The name of an extension member or extension type MUST be a reverse
domain name under the control of the defining party, such as
`com.example.gpuUtilization`. Receivers MUST ignore members and metric sets that
they do not understand.


# Metric Reports {#report}

A Metric Report carries metric sets from one reporter. It has the following
fields:

`version`:
: The version of this format. For this document, `draft-00`.

`reporter`:
: A map containing the reporter `id` and its `role`: `publisher`,
  `subscriber`, or `relay`.

`generatedAt`:
: The time the report was generated, in milliseconds since the Unix epoch.

`sequence`:
: OPTIONAL. A number that increases by one with each report from the reporter,
  allowing a collector to detect missing reports.

`sets`:
: A list of metric sets.

A reporter SHOULD include in each report every metric set that has changed since
the previous report. Because counters are cumulative, a lost report reduces the
time resolution of the data but does not lose counts.

## Encodings

A Metric Report is encoded in JSON {{JSON}} or in CBOR {{CBOR}}. Both encodings
use the same data model and the same field names, as specified by the CDDL
{{CDDL}} in {{cddl}}. The media types `application/moq-analytics+json` and
`application/moq-analytics+cbor` identify the two encodings.

JSON is easy to inspect and matches the encoding of the MSF catalog. CBOR is
more compact and is RECOMMENDED for analytics tracks that report frequently or
that share capacity with media.

CBOR was chosen over schema-dependent binary formats such as Protocol Buffers
{{PROTOBUF}} because CBOR is self-describing, so a receiver can skip members it
does not understand without a schema, and because a single CDDL specification
describes both the JSON and CBOR encodings.

## Example {#example}

The following JSON Metric Report contains three metric sets from a subscriber:

~~~ json
{
  "version": "draft-00",
  "reporter": { "id": "c7f3a1", "role": "subscriber" },
  "generatedAt": 1790438400250,
  "sequence": 42,
  "sets": [
    {
      "type": "transport",
      "id": "T1",
      "timestamp": 1790438400200,
      "bytesReceived": 48213377,
      "packetsLost": 112,
      "smoothedRtt": 0.031,
      "availableBitrate": 1800000
    },
    {
      "type": "inbound-track",
      "id": "IT1",
      "timestamp": 1790438400200,
      "trackName": "video",
      "deliveryMode": "subgroup",
      "objectsReceived": 5412,
      "objectsLost": 3,
      "largestLocation": { "group": 180, "object": 11 },
      "latency": {
        "count": 5412,
        "sum": 437.9,
        "bounds": [0.05, 0.1, 0.25, 0.5],
        "counts": [4101, 1122, 170, 16, 3]
      }
    },
    {
      "type": "playback",
      "id": "P1",
      "timestamp": 1790438400200,
      "trackIds": ["IT1"],
      "state": "playing",
      "bufferLevel": 1.35,
      "liveLatency": 2.1,
      "stallCount": 1,
      "totalStallTime": 0.42
    }
  ]
}
~~~
{: title="Example Metric Report"}


# Delivery over MoQT {#delivery}

## Analytics Tracks {#analytics-tracks}

A reporter delivers Metric Reports by publishing them as objects on an
analytics track. Analytics tracks are ordinary MoQT tracks, so subscription,
relay forwarding, caching, and FETCH all follow {{MOQT}}. No report-level
routing fields are needed.

Analytics tracks map Metric Reports onto MoQT objects as follows:

* Each group carries the reports generated at one time. The Group ID SHOULD be
  the `generatedAt` value of the reports in the group, consistent with the MSF
  metrics track. Group IDs MUST increase.
* Each object carries exactly one complete Metric Report. A reporter MAY split
  the metric sets generated at one time across several objects in the same
  group, for example to limit object size. Each such object MUST be a complete
  Metric Report that can be decoded independently.
* The encoding is identified by the catalog ({{catalog}}) or by configuration
  ({{configured}}). All objects on a track use the same encoding.

Analytics are usually less urgent than media. A reporter SHOULD assign analytics
tracks a publisher priority that schedules media objects ahead of analytics
objects on the same session, so that analytics use capacity that media leaves
unused. A reporter MAY use the delivery timeout mechanism of {{MOQT}} to discard
reports that are no longer timely, and MAY send small reports as datagrams.

When MSF is in use, a reporter MAY compress objects using the compression
signaling defined in {{MSF}}.

## Catalog-Directed Delivery {#catalog}

An MSF {{MSF}} or CMSF {{CMSF}} catalog requests analytics by declaring an
analytics track in its `publishTracks` array. The track object:

* MUST have a `packaging` value of `moqanalytics`;
* MUST have a `role` value of `metrics`;
* MUST have a `mimeType` value identifying the encoding; and
* MAY include the `connectionUri` and `token` fields defined by {{MSF}}, to
  direct reports to a collector other than the catalog's origin and to
  authorize publishing.

This document defines two additional track object fields:

`metricSets`:
: An array of metric set types the subscriber is asked to report. A subscriber
  SHOULD report the requested sets that it can measure, and SHOULD NOT report
  other sets.

`reportInterval`:
: The requested time between reports, in integer milliseconds.

The `namespace` and `name` fields give the analytics track. The `namespace`
MAY contain the placeholder `%reporterId%`, which the subscriber replaces with
its reporter `id`.

~~~ json
"publishTracks": [
  {
    "namespace": "analytics.example.com/v1/%reporterId%",
    "name": "qoe",
    "packaging": "moqanalytics",
    "role": "metrics",
    "mimeType": "application/moq-analytics+cbor",
    "connectionUri": "moqt://collector.example.com:4443",
    "token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...",
    "metricSets": ["transport", "inbound-track", "playback"],
    "reportInterval": 2000
  }
]
~~~
{: title="Catalog Requesting Analytics"}

Because CMSF uses the MSF catalog, the same declaration applies to CMSF
catalogs. A catalog MAY declare several analytics tracks, for example one with
a short interval for QoE metrics and one with a long interval for transport
metrics.

## Reporter-Configured Delivery {#configured}

A reporter MAY be configured by its application or operator with a collector
endpoint, credentials, an analytics track, an encoding, metric sets, and a
reporting interval. The reporter establishes a MoQT session to the collector,
if needed, and publishes the analytics track as described in
{{analytics-tracks}}.

This mode supports deployments that do not use MSF, and lets publishers and
relays report to an operator's collector.

## Collector Subscription

A relay or publisher MAY publish its analytics track continuously within a
namespace that collectors can subscribe to. Collectors then subscribe only to
the reporters and tracks they need, and relays cache recent reports for later
retrieval with FETCH. This allows metrics to be pulled on demand rather than
pushed to every collector.


# Export to Monitoring Systems {#export}

An implementation MAY export metric sets to OpenMetrics {{OPENMETRICS}},
OpenTelemetry {{OTEL}}, or another monitoring system. To keep metric names
consistent across implementations, an exported metric name SHOULD be formed by
joining the following with underscores:

1. the prefix `moq`;
2. the metric set type, with hyphens replaced by underscores;
3. the member name, converted from lowerCamelCase to lowercase words separated
   by underscores; and
4. for counters, the suffix `total`.

For example, `objectsLost` in an `inbound-track` set is exported as
`moq_inbound_track_objects_lost_total`. Histograms map to explicit bucket
histograms. Identifying attributes such as `kind` or `trackName` MAY be exported
as labels, subject to the guidance in {{cardinality}}.

qlog {{MOQ-QLOG}} is better suited than metric sets for event traces and
protocol debugging. An implementation MAY correlate metric sets with qlog
events using a session identifier.


# Mappings from Existing Vocabularies

## WebRTC Statistics

The metric sets in this document follow the model of the WebRTC statistics API
{{WEBRTC-STATS}}, and media members share names with WebRTC statistics where
the definitions match. The following WebRTC statistics types correspond to
metric set types:

| WebRTC type | Metric set type |
|---|---|
| `outbound-rtp`, `media-source` | `media-source` |
| `inbound-rtp` | `inbound-media` |
| `candidate-pair`, `transport` | `transport` |
| `codec` | `codec` member of media sets |
{: title="WebRTC Statistics Types"}

The following members differ from their WebRTC counterparts:

* `currentRoundTripTime` corresponds to `latestRtt` or `smoothedRtt`, depending
  on how the implementation computes it;
* `availableOutgoingBitrate` corresponds to `availableBitrate`;
* `frameWidth` and `frameHeight` correspond to `width` and `height`; and
* RTP `jitter` is not equivalent to object `interarrival` in an
  `inbound-track` set.

A MoQT implementation MUST NOT report a member derived from a WebRTC statistic
that it has not collected at the corresponding point.

## CMCD and CMSD

Common Media Client Data {{CMCD}} and Common Media Server Data {{CMSD}} carry
client and server delivery information in HTTP adaptive streaming. MoQT has no
HTTP requests or responses, so their keys are not carried directly. The
following correspondence is informative:

| Key | Concept | Member |
|---|---|---|
| CMCD `bl` | buffer length | `playback.bufferLevel` |
| CMCD `bs` | buffer starvation | `playback.stallCount` |
| CMCD `dl` | deadline | `playback.deadline` |
| CMCD `mtp` | measured throughput | `inbound-media.bitrate` |
| CMCD `su` | startup | `playback.state` |
| CMSD `etp` | estimated throughput | `transport.availableBitrate` |
| CMSD `rd` | response delay | `relay.processingTime` |
| CMSD `ht` | held time | `relay.heldTime` |
| CMSD `at` | availability time | `relay.availabilityTime` |
| CMSD `du` | duress | `relay.duress` |
{: title="CMCD and CMSD Correspondence"}

CMCD measured throughput is a client measurement of achieved throughput,
whereas CMSD estimated throughput is a server estimate. A CMSD maximum
suggested bitrate is a recommendation rather than a measurement and has no
corresponding member.


# Security Considerations

Metric Reports can influence operational decisions such as rate adaptation,
relay selection, and capacity planning, and are therefore targets for
spoofing, injection, and replay.

A Metric Report MUST NOT be assumed to be accurate merely because it arrived
over an authenticated MoQT session. A subscriber can report incorrect playback
state, and a relay can report incorrect operational state. Collectors SHOULD
treat reports as claims made by the named reporter.

A catalog that includes a `connectionUri` directs subscribers to connect to
another endpoint. A subscriber MUST NOT follow a `connectionUri` from a catalog
that it did not receive from an authenticated, authorized source, since
otherwise an attacker could direct subscribers to connect to arbitrary hosts.
Tokens in the catalog SHOULD be scoped to the analytics track and have a short
lifetime.

Collectors and relays SHOULD limit the size and rate of Metric Reports and the
number of distinct metric sets accepted from each reporter, to limit resource
exhaustion. Relays SHOULD apply the same authorization to analytics tracks as
to other tracks. When relays must not read analytics, the reporter can protect
objects end to end using {{SECURE-OBJECTS}}.


# Privacy Considerations

Metrics can reveal track names, viewing behavior, network characteristics,
device capabilities, and topology, and can be used to identify or track users.

A reporter `id` SHOULD NOT be derived from a stable hardware identifier, such as
a network interface hardware address, and subscribers SHOULD use a reporter `id`
that is not linkable across sessions unless the user has agreed otherwise.
Reporters SHOULD report track names only when they are needed
({{cardinality}}). Collectors SHOULD aggregate subscriber metrics where
individual detail is not needed, and SHOULD document retention policies for
subscriber metrics.


# IANA Considerations

This document has no IANA actions. A future revision is expected to request:

* registration of the media types `application/moq-analytics+json` and
  `application/moq-analytics+cbor`;
* registration of the MSF packaging value `moqanalytics`, if MSF establishes a
  registry for packaging values; and
* a registry of metric set types and members.


# Use of Generative AI
{:numbered="false"}

Generative AI tools were used to assist with drafting and editing text for this
document. All AI-generated content was reviewed and approved by the author.


--- back

# CDDL {#cddl}

The following CDDL {{CDDL}} specifies the Metric Report for both the JSON and
CBOR encodings.

~~~ cddl
metric-report = {
  version: tstr,
  reporter: reporter,
  generatedAt: timestamp,
  ? sequence: uint,
  sets: [* metric-set],
  * ext-key => any,
}

reporter = {
  id: tstr,
  role: "publisher" / "subscriber" / "relay",
  * ext-key => any,
}

; Extension keys are reverse domain names, e.g., "com.example.fooBar"
ext-key = tstr .regexp ext-re
ext-re = "[a-z0-9-]+([.][a-z0-9-]+)+[.][A-Za-z][A-Za-z0-9-]*"

timestamp = number   ; milliseconds since the Unix epoch
seconds = number     ; duration or cumulative time
counter = uint       ; cumulative over the lifetime of the set
bps = number         ; bits per second

histogram = {
  count: uint,
  sum: number,
  ? min: number,
  ? max: number,
  ? bounds: [* number],  ; ascending upper bounds
  ? counts: [* uint],    ; one more entry than bounds
}

location = {
  group: uint,
  object: uint,
}

set-common<T> = (
  type: T,
  id: tstr,
  timestamp: timestamp,
)

track-ref = (
  ? namespace: [* tstr],
  ? trackName: tstr,
)

metric-set = session-set / requests-set / transport-set /
             outbound-track-set / inbound-track-set /
             subscription-set / media-source-set /
             inbound-media-set / playback-set / relay-set /
             extension-set

session-set = {
  set-common<"session">,
  ? transportId: tstr,
  ? version: tstr,
  ? binding: "quic" / "webtransport",
  ? state: "connecting" / "established" / "draining" / "closed",
  ? setupTime: seconds,
  ? goawaysReceived: counter,
  * ext-key => any,
}

requests-set = {
  set-common<"requests">,
  kind: "subscribe" / "fetch" / "publish" / "publish-namespace" /
        "subscribe-namespace" / "track-status" / tstr,
  ? sessionId: tstr,
  ? sent: counter,
  ? received: counter,
  ? succeeded: counter,
  ? failed: counter,
  ? active: uint,
  ? responseTime: histogram,
  ? errorCodes: [* [code: uint, count: counter]],
  * ext-key => any,
}

transport-set = {
  set-common<"transport">,
  ? sessionId: tstr,
  ? bytesSent: counter,
  ? bytesReceived: counter,
  ? packetsSent: counter,
  ? packetsReceived: counter,
  ? packetsLost: counter,
  ? latestRtt: seconds,
  ? minRtt: seconds,
  ? smoothedRtt: seconds,
  ? rttVar: seconds,
  ? cwnd: uint,
  ? bytesInFlight: uint,
  ? availableBitrate: bps,
  ? cwndBlockedTime: seconds,
  ? flowBlockedTime: seconds,
  ? streamsOpened: counter,
  ? streamsActive: uint,
  ? datagramsSent: counter,
  ? datagramsReceived: counter,
  ? datagramsLost: counter,
  * ext-key => any,
}

delivery-mode = "subgroup" / "datagram" / "mixed"

outbound-track-set = {
  set-common<"outbound-track">,
  track-ref,
  ? sessionId: tstr,
  ? deliveryMode: delivery-mode,
  ? priority: uint,
  ? objectsSent: counter,
  ? bytesSent: counter,
  ? groupsSent: counter,
  ? objectsDropped: counter,
  ? streamsReset: counter,
  ? subscriptions: uint,
  ? largestLocation: location,
  * ext-key => any,
}

inbound-track-set = {
  set-common<"inbound-track">,
  track-ref,
  ? sessionId: tstr,
  ? deliveryMode: delivery-mode,
  ? objectsReceived: counter,
  ? bytesReceived: counter,
  ? groupsReceived: counter,
  ? groupsCompleted: counter,
  ? objectsLost: counter,
  ? objectsLate: counter,
  ? duplicates: counter,
  ? streamsReset: counter,
  ? largestLocation: location,
  ? interarrival: histogram,
  ? latency: histogram,
  ? groupDuration: histogram,
  * ext-key => any,
}

subscription-set = {
  set-common<"subscription">,
  track-ref,
  ? sessionId: tstr,
  ? trackId: tstr,
  ? requestId: uint,
  ? state: "pending" / "active" / "done" / "error",
  ? forward: bool,
  ? priority: uint,
  ? groupOrder: "ascending" / "descending",
  ? responseTime: seconds,
  ? repairRequests: counter,
  * ext-key => any,
}

media-kind = "audio" / "video" / tstr

media-source-set = {
  set-common<"media-source">,
  ? trackId: tstr,
  ? kind: media-kind,
  ? codec: tstr,
  ? bitrate: bps,
  ? framesEncoded: counter,
  ? keyFramesEncoded: counter,
  ? framesPerSecond: number,
  ? keyFrameInterval: seconds,
  ? width: uint,
  ? height: uint,
  ? totalEncodeTime: seconds,
  * ext-key => any,
}

inbound-media-set = {
  set-common<"inbound-media">,
  ? trackId: tstr,
  ? kind: media-kind,
  ? codec: tstr,
  ? bitrate: bps,
  ? framesReceived: counter,
  ? framesDecoded: counter,
  ? keyFramesDecoded: counter,
  ? framesDropped: counter,
  ? framesPerSecond: number,
  ? totalDecodeTime: seconds,
  ? jitter: seconds,
  ? width: uint,
  ? height: uint,
  ? concealedSamples: counter,
  * ext-key => any,
}

playback-set = {
  set-common<"playback">,
  ? trackIds: [* tstr],
  ? renderGroup: uint,
  ? state: "startup" / "playing" / "paused" / "stalled" / "ended",
  ? startupTime: seconds,
  ? bufferLevel: seconds,
  ? targetLatency: seconds,
  ? liveLatency: seconds,
  ? deadline: seconds,
  ? stallCount: counter,
  ? totalStallTime: seconds,
  ? nonRendered: counter,
  ? switches: counter,
  * ext-key => any,
}

relay-set = {
  set-common<"relay">,
  ? sessions: uint,
  ? subscriptions: uint,
  ? fetches: uint,
  ? cacheHits: counter,
  ? cacheMisses: counter,
  ? cacheBytes: uint,
  ? processingTime: histogram,
  ? heldTime: histogram,
  ? availabilityTime: histogram,
  ? duress: number,
  * ext-key => any,
}

extension-set = {
  set-common<ext-key>,
  * tstr => any,
}
~~~
{: title="Metric Report CDDL"}

# Open Issues
{:removeinrfc="true"}

The following questions are open for discussion:

1. Which metric sets and members belong in a base set that all reporters are
   expected to support?
2. Should the CBOR encoding assign integer keys to fields and members to reduce
   size further?
3. Should histograms and counters support delta temporality in addition to
   cumulative temporality?
4. Should analytics tracks use a new MSF packaging value, as proposed here, or a
   profile of the `moqmetrics` packaging of {{MOQMETRICS}}? MSF currently lists
   packaging values without a registry.
5. Should analytics track names follow the granularity levels used by MSF
   metrics tracks?
6. How should track namespaces, which are tuples of byte strings in {{MOQT}}, be
   represented when they are not valid text?
7. Which members of the feedback reports in {{MOQFEEDBACK}} should be aligned
   with members defined here?
8. What clock synchronization should be assumed for `latency` and
   `liveLatency`?

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
