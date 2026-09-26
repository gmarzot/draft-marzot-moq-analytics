---
title: "Metrics and Analytics for Media over QUIC Transport"
abbrev: "MoQT Analytics"
category: info

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
    email: giovani.marzot@oracle.com

normative:
  MOQT: I-D.ietf-moq-transport

informative:
  MSF: I-D.ietf-moq-msf
  LOC: I-D.ietf-moq-loc
  CMSF: I-D.ietf-moq-cmsf
  MOQFEEDBACK: I-D.liu-moq-feedback
  MOQMETRICS: I-D.jennings-moq-metrics
  MOQ-QLOG: I-D.pardue-moq-qlog-moq-events
  SECURE-OBJECTS: I-D.ietf-moq-secure-objects
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

--- abstract

Media over QUIC Transport (MoQT) deployments need metrics that can be
interpreted consistently across publishers, subscribers, relays, and
collectors. Existing systems expose transport counters, media statistics,
playback statistics, and request outcomes using different names, scopes, and
units.

This document defines a MoQT-oriented vocabulary and reporting model for
publisher-, subscriber-, relay-, session-, link-, namespace-, and track-level
metrics. It defines scope and cardinality guidance, a small common set of metric
types and units, and informative mappings from metrics commonly available in
WebRTC statistics, CMCD, and CMSD. It also describes how the vocabulary can be
carried using existing or companion mechanisms, including MSF metrics tracks,
subscriber feedback tracks, Event Timeline tracks, dedicated MoQT tracks, and
implementation-specific export formats.

This document does not define a mandatory metrics transport, a universal
playback model, or a replacement for QUIC congestion-control feedback.

--- middle

# Introduction

MoQT {{MOQT}} is a media-agnostic publish/subscribe protocol running over QUIC
and WebTransport. It exposes protocol events and delivery coordinates, such as
sessions, requests, namespaces, tracks, groups, objects, streams, and datagrams,
that are useful for operational measurement. A deployment may also have a media
pipeline that exposes codec, frame, buffer, and playback statistics.

These measurements are currently difficult to combine. One implementation may
expose subscription counters, another may expose QUIC recovery state, and a
media endpoint may expose WebRTC statistics {{WEBRTC-STATS}}. The same concept
may have different names, units, labels, and aggregation rules. A relay operator
therefore cannot reliably compare a publisher's observations with a subscriber's
observations or correlate them with relay and transport events.

This document defines a common vocabulary rather than prescribing a single
monitoring system. An implementation can expose the metrics through OpenMetrics
{{OPENMETRICS}}, OpenTelemetry {{OTEL}}, qlog {{MOQ-QLOG}}, a MoQT metrics track,
an MSF {{MSF}} track, a feedback track, or another format. The metric definition
remains separate from the carriage mechanism.

## Goals

This document has four goals:

1. define MoQT-native scopes and terminology for measurements;
2. identify a small interoperable set of metrics with explicit type, unit, and
   aggregation semantics;
3. provide practical mappings from statistics that endpoint implementations
   commonly expose today; and
4. allow the same metric semantics to be carried through different MoQT
   packaging and reporting mechanisms.

## Non-Goals

This document does not require a node to expose every metric. It distinguishes
metrics that are directly observable at the MoQT or QUIC layer from metrics that
require a media pipeline, decoder, renderer, or application-specific deadline.

This document does not define a wire encoding for metric reports, an alternate
reporting destination, or relay forwarding behavior for reports. These are left
to carriage documents (see {{carriage}}).

CMCD {{CMCD}} and CMSD {{CMSD}} are treated as related HTTP adaptive-streaming
vocabularies: their concepts inform the mapping of client QoE and server
delivery metrics, while their HTTP carriage is not assumed by MoQT.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms, in addition to those defined in
{{MOQT}}:

Publisher:
: An endpoint that publishes a namespace or track and sends objects.

Subscriber:
: An endpoint that subscribes to or fetches a track and receives objects.

Relay:
: A MoQT intermediary that forwards or caches objects between sessions.

Reporter:
: A publisher, subscriber, relay, or collector-side component that creates a
  measurement.

Collector:
: A component that receives, stores, aggregates, or presents measurements.

Session:
: One MoQT connection and its negotiated protocol state.

Link:
: One directed observation path over a session, normally between two directly
  connected MoQT nodes.

Namespace:
: A MoQT track namespace.

Track:
: A MoQT track identified by a namespace and track name.

Group:
: A sequence of objects with a common group identifier.

Object:
: A MoQT delivery unit identified by a group and object identifier.

Delivery Mode:
: The mechanism used to carry objects, such as a subgroup stream, a per-object
  stream, or a datagram.

QoE:
: Application or playback observations that describe user-visible quality.
  QoE metrics are not inferred from transport measurements unless explicitly
  stated.

Metric names in this document use the `moq_` prefix and snake case.
Implementations MAY expose equivalent names in another namespace, but a mapping
to the names and semantics in this document is necessary for interoperability.


# Design Principles

## Measurement Layers {#layers}

Measurements are divided into layers:

Application and request layer:
: Request outcomes, request latency, authorization results, and session
  lifecycle.

MoQT delivery layer:
: Object counts, bytes, group boundaries, delivery mode, object locations,
  stream outcomes, and subscription state.

QUIC transport layer:
: RTT, loss, congestion-control state, bytes, streams, flow control, and
  datagram state.

Media pipeline layer:
: Codec, encoded and decoded frames, frame rate, keyframes, decode time, render
  state, and playout buffer.

Playback and QoE layer:
: Startup, stalls, live latency, deadline headroom, dropped frames, and
  non-rendered content.

A metric MUST identify the layer at which it is measured or derived. A
subscriber MUST NOT report an application-layer playback metric as if it were a
QUIC or MoQT-layer observation.

## Observation versus Interpretation

A metric definition describes an observation and its unit. Implementations MAY
derive higher-level indicators from multiple observations, but the derivation
SHOULD be documented. For example, a receiver can report
`moq_track_objects_received_total` from its MoQT object callback. It cannot
infer `moq_sub_buffer_starvation_total` without a playback or rendering signal.

## Metric Types {#types}

Counter:
: A cumulative count that increases during a process lifetime and may reset when
  the reporting process restarts.

Gauge:
: A sampled value that may increase or decrease.

Histogram:
: A distribution of observations. A histogram has an observation unit and an
  aggregation policy; bucket boundaries are a collector or profile choice unless
  a metric definition specifies them.

An implementation exporting a counter MUST document reset behavior. A collector
MUST use the reporter identity and process or session lifetime when interpreting
counter resets.


# MoQT Observability Model

## Scopes

A deployment can be viewed as the following hierarchy:

~~~ ascii-art
Deployment
  -> Relay Mesh
    -> Relay Node
      -> Link
        -> Namespace
          -> Track
            -> Group
              -> Object
~~~
{: title="Observation Scopes"}

This hierarchy describes possible scopes, not a requirement that every report
contain every ancestor. A report SHOULD carry enough context for a collector to
associate it with a deployment, node, session, namespace, or track according to
local authorization policy.

The principal operational scope is normally the track. Group and object
identifiers are useful for local correlation and debugging, but they are usually
unsuitable as metric labels. Implementations SHOULD aggregate group- and
object-level observations into counters or histograms and SHOULD avoid
unbounded labels containing object identifiers.

## Reporter Perspective

The same concept can differ by vantage point. For example:

* a publisher's sent bitrate is measured before or at publisher-side MoQT
  transmission;
* a relay's forwarded bitrate is measured on a particular link;
* a subscriber's received bitrate is measured after object admission at the
  receiving MoQT layer; and
* a player's rendered bitrate is measured after decode and playout selection.

Metric names SHOULD distinguish these perspectives through scope or an explicit
reporter role. A receiver MUST NOT use a publisher-side name for a
subscriber-side measurement merely because both values are expressed in the
same unit.

## Labels and Cardinality {#labels}

Labels are dimensions of a time series, not a replacement for report scope.
Implementations SHOULD prefer a bounded label set such as:

* `role`: `publisher`, `subscriber`, `relay`, or `collector`;
* `direction`: `upstream` or `downstream`;
* `result`: a stable result class and, where useful, a protocol error class;
* `delivery_mode`: `subgroup`, `stream`, or `datagram`;
* `version`: the negotiated MoQT version or protocol profile; and
* a deployment-controlled relay or link identifier.

Namespace and track labels MAY be used where their cardinality and privacy
impact are acceptable. Implementations SHOULD NOT use raw object identifiers,
request identifiers, subscriber identities, or unconstrained URLs as labels in
fleet-wide time series. Such values MAY appear in a bounded diagnostic report
or trace event when authorized.


# Metric Registry {#registry}

This section defines a starter set of metrics. Each entry has a name, type,
unit, scope, and description.

An implementation MAY omit a metric when the observation is unavailable, but
MUST NOT emit a value that it cannot measure or derive as specified.

## Session and Request Metrics

`moq_sessions_active`:
: Gauge. Unit: sessions. Scope: node. Active MoQT sessions.

`moq_session_duration_seconds`:
: Histogram. Unit: seconds. Scope: session. Duration of completed sessions.

`moq_requests_total`:
: Counter. Unit: requests. Scope: node, session. Requests by request kind and result class.

`moq_request_duration_seconds`:
: Histogram. Unit: seconds. Scope: request kind. Time from request creation to terminal response.

`moq_request_errors_total`:
: Counter. Unit: errors. Scope: request kind. Requests ending in an error, optionally classified by stable error class.

`moq_subscriptions_active`:
: Gauge. Unit: subscriptions. Scope: node, namespace, track. Active subscriptions at the reporting node.

`moq_fetches_active`:
: Gauge. Unit: fetches. Scope: node, namespace, track. Active FETCH operations at the reporting node.

The request kind SHOULD be one of `subscribe`, `fetch`, `publish`,
`publish_namespace`, `subscribe_namespace`, `track_status`, or an extension
defined by the relevant protocol document.

## Link and QUIC Metrics

`moq_link_rtt_seconds`:
: Gauge or histogram. Unit: seconds. Scope: link. Smoothed or sampled round-trip time, with the chosen QUIC statistic identified.

`moq_link_available_bitrate_bps`:
: Gauge. Unit: bits/s. Scope: link. Estimated available bitrate, when exposed by the transport or media stack.

`moq_link_packet_loss_ratio`:
: Gauge. Unit: ratio. Scope: link. Packet loss ratio over a defined observation interval.

`moq_link_bytes_sent_total`:
: Counter. Unit: bytes. Scope: link. Bytes submitted to the transport in the reporting direction.

`moq_link_bytes_received_total`:
: Counter. Unit: bytes. Scope: link. Bytes admitted from the transport in the reporting direction.

`moq_link_streams_active`:
: Gauge. Unit: streams. Scope: link. Active MoQT or QUIC streams, with stream class if bounded.

`moq_link_datagrams_sent_total`:
: Counter. Unit: datagrams. Scope: link. Datagrams submitted.

`moq_link_datagrams_lost_total`:
: Counter. Unit: datagrams. Scope: link. Datagrams inferred or reported as lost.

`moq_link_congestion_blocked_seconds_total`:
: Counter. Unit: seconds. Scope: link. Time during which application writes were blocked by the transport or configured send budget.

`moq_link_cwnd_bytes`:
: Gauge. Unit: bytes. Scope: link. Congestion window, only when exposed by the transport implementation.

`moq_link_flow_control_blocked_seconds_total`:
: Counter. Unit: seconds. Scope: link. Time blocked by flow-control limits, distinct from congestion-control blocking where distinguishable.

QUIC measurements are implementation-specific unless a QUIC specification such
as {{?RFC9000}} or {{?RFC9002}} defines the statistic. An implementation SHOULD
identify whether a value comes from QUIC recovery, a transport binding, or an
application-side observation.

## MoQT Delivery Metrics

`moq_track_objects_sent_total`:
: Counter. Unit: objects. Scope: track, link. Objects submitted for transmission.

`moq_track_objects_received_total`:
: Counter. Unit: objects. Scope: track, link. Objects admitted at the receiving MoQT layer.

`moq_track_objects_dropped_total`:
: Counter. Unit: objects. Scope: track, link. Objects discarded by a configured local policy, with the policy distinguished from transport loss where possible.

`moq_track_bytes_sent_total`:
: Counter. Unit: bytes. Scope: track, link. Object payload or object wire bytes sent, with the counted convention documented.

`moq_track_bytes_received_total`:
: Counter. Unit: bytes. Scope: track, link. Object payload or object wire bytes received, with the counted convention documented.

`moq_track_delivery_latency_seconds`:
: Histogram. Unit: seconds. Scope: track, link. Sender-to-receiver latency when an object timestamp or synchronized measurement makes it meaningful.

`moq_track_object_interarrival_seconds`:
: Histogram. Unit: seconds. Scope: track. Inter-arrival interval of admitted objects.

`moq_track_groups_completed_total`:
: Counter. Unit: groups. Scope: track. Groups observed or completed according to the local delivery rule.

`moq_track_group_duration_seconds`:
: Histogram. Unit: seconds. Scope: track. Time span between the first and last object in a group, when defined.

`moq_track_delivery_mode`:
: Enumerated attribute. Scope: track. Delivery mode used by the track or subscription.

`moq_track_priority`:
: Gauge. Unit: priority value. Scope: track, link. MoQT priority value applied to the relevant request or delivery.

`moq_subscription_objects_late_total`:
: Counter. Unit: objects. Scope: subscription. Objects arriving after an application-defined deadline.

`moq_subscription_objects_lost_total`:
: Counter. Unit: objects. Scope: subscription. Objects classified as not received or otherwise unavailable.

`moq_subscription_repair_requests_total`:
: Counter. Unit: requests. Scope: subscription. FETCH or equivalent repair requests issued for the subscription.

The definitions of "sent", "received", "dropped", and "lost" MUST identify the
observation point. A receiver MUST NOT count a missing object as lost solely
because it has not yet reached a report deadline.

## Media and Subscriber QoE Metrics

These metrics require a media pipeline, decoder, renderer, or
application-specific playback model. They cannot be produced by a generic MoQT
transport alone.

`moq_media_encoded_bitrate_bps`:
: Gauge. Unit: bits/s. Scope: publisher, track. Encoded media bitrate.

`moq_media_received_bitrate_bps`:
: Gauge. Unit: bits/s. Scope: subscriber, track. Media bytes received over a defined interval.

`moq_media_frame_rate_fps`:
: Gauge. Unit: frames/s. Scope: track. Encoded, decoded, or rendered frame rate; the stage MUST be stated.

`moq_media_frames_encoded_total`:
: Counter. Unit: frames. Scope: publisher, track. Frames encoded by the media pipeline.

`moq_media_frames_decoded_total`:
: Counter. Unit: frames. Scope: subscriber, track. Frames decoded by the media pipeline.

`moq_media_keyframes_encoded_total`:
: Counter. Unit: frames. Scope: publisher, track. Keyframes or random-access frames encoded.

`moq_media_keyframe_interval_seconds`:
: Gauge or histogram. Unit: seconds. Scope: track. Interval between keyframes or random-access points.

`moq_media_codec`:
: Enumerated attribute. Scope: track. Codec and profile at the media pipeline boundary.

`moq_media_jitter_seconds`:
: Gauge or histogram. Unit: seconds. Scope: subscriber, track. Media-pipeline jitter; distinct from MoQT object inter-arrival jitter.

`moq_media_decode_time_seconds`:
: Histogram. Unit: seconds. Scope: subscriber, track. Time spent decoding media samples or frames.

`moq_sub_buffer_level_seconds`:
: Gauge. Unit: seconds. Scope: subscriber, track. Forward buffer or playout buffer level, according to the application definition.

`moq_sub_buffer_starvation_total`:
: Counter. Unit: events. Scope: subscriber, track. Buffer starvation or rebuffering events.

`moq_sub_startup_delay_seconds`:
: Histogram. Unit: seconds. Scope: subscriber, track. Time from the selected startup event to first usable or rendered media.

`moq_sub_live_latency_seconds`:
: Gauge or histogram. Unit: seconds. Scope: subscriber, track. End-to-end live latency, requiring an agreed clock or timeline.

`moq_sub_deadline_seconds`:
: Gauge. Unit: seconds. Scope: subscriber, track. Application-defined time until the next playback deadline.

`moq_sub_dropped_frames_total`:
: Counter. Unit: frames. Scope: subscriber, track. Frames dropped by the media pipeline or renderer.

`moq_sub_non_rendered_total`:
: Counter. Unit: objects or frames. Scope: subscriber, track. Media received or decoded but not rendered, with the cause classified where possible.

`moq_sub_playout_ahead_seconds`:
: Gauge. Unit: seconds. Scope: subscriber, track. Time remaining before the playout buffer is exhausted.

## Relay and Resource Metrics

`moq_relay_request_processing_seconds`:
: Histogram. Unit: seconds. Scope: relay, request kind. Relay processing time excluding downstream transport delay where measurable.

`moq_relay_held_seconds`:
: Histogram. Unit: seconds. Scope: relay, request kind. Time a request or object was held by a relay policy or queue.

`moq_relay_object_availability_seconds`:
: Histogram. Unit: seconds. Scope: relay, track. Time between object availability at the relay and the relay's forwarding decision or admission.

`moq_relay_cache_hits_total`:
: Counter. Unit: objects or groups. Scope: relay, track. Cache hits, with the cache lookup unit specified.

`moq_relay_cache_misses_total`:
: Counter. Unit: objects or groups. Scope: relay, track. Cache misses.

`moq_relay_duress`:
: Gauge. Unit: boolean or ratio. Scope: relay. Relay load or duress indicator; the definition MUST be deployment-specific and documented.

`moq_relay_suggested_bitrate_bps`:
: Gauge. Unit: bits/s. Scope: relay, track. A relay-originated bitrate recommendation, if a companion control mechanism defines one.


# Report Data Model {#report}

A metrics export MAY represent a single sample, a batch of samples, or a time
series. When a structured report is used, the following conceptual fields are
RECOMMENDED:

~~~ pseudocode
MetricReport {
  report_time   : timestamp,
  reporter_id   : opaque identifier,
  reporter_role : publisher | subscriber | relay | collector,
  session_id    : opaque identifier (optional),
  scope         : deployment | relay | link | session | namespace |
                  track | subscription,
  scope_id      : opaque identifier,
  samples       : [MetricSample],
  sequence      : unsigned integer (optional),
  expires       : timestamp or duration (optional)
}

MetricSample {
  name          : registered metric name,
  type          : counter | gauge | histogram,
  unit          : registered unit,
  value         : typed metric value,
  attributes    : bounded key/value set (optional),
  interval      : observation interval (optional),
  source        : observation layer or source (optional)
}
~~~
{: title="Conceptual Report Structure"}

The structured model is intentionally conceptual. A carriage document MAY use a
binary, JSON, CBOR, catalog-defined, or metrics-system-native encoding. It MUST
define how metric type, unit, timestamp, reset, and observation interval are
represented.

Unknown metric names MAY be ignored. Unknown fields in an extensible report
SHOULD be ignored unless the carriage document specifies a stricter rule. A
receiver SHOULD preserve unknown samples when forwarding an opaque report.

This document does not define a destination URI or relay forwarding behavior
for reports. A carriage or authorization document that permits a report to
leave the delivery path MUST define destination authorization, loop prevention,
replay handling, and privacy requirements. Those semantics cannot be inferred
from a report field alone.


# Carriage and Integration {#carriage}

## Metrics Tracks and MSF

MSF {{MSF}} defines catalog and track conventions for logs and metrics tracks,
including a packaging value and a resource-oriented metrics payload derived from
{{MOQMETRICS}}. This document does not replace that payload. A future revision
MAY define a profile or mapping that carries this registry in an MSF metrics
track.

An implementation using an MSF metrics track SHOULD identify:

* the associated media or session scope;
* the reporter role and reporting direction;
* whether the payload is a registry sample, a time series, or a feedback report;
  and
* the lifetime, authorization, and intended collector of the metrics track.

## Feedback Tracks

A feedback track is a reporting mechanism, not a complete metrics ontology. For
example, {{MOQFEEDBACK}} defines per-object delivery statuses and summary
metrics carried in feedback tracks. The registry in this document can provide
names and semantics for optional summary fields, but it MUST NOT be used to
redefine a feedback track's lifecycle, routing, or per-object encoding.

A feedback report SHOULD distinguish:

* transport feedback used by congestion control;
* MoQT delivery feedback used by a publisher or relay; and
* application or playback feedback used by an encoder, agent, or player.

## Event Timeline and Packaging-Specific Tracks

MSF Event Timeline tracks provide a carrier for event-oriented telemetry, such
as CMCD reports. An Event Timeline payload identifies its own event type and
indexing rules. This document defines reusable metric semantics; it does not
require all events to be converted into scalar metrics.

LOC {{LOC}} and CMSF {{CMSF}} MAY define packaging-specific metrics. Such
metrics SHOULD follow the scope, type, unit, and cardinality principles of this
document where applicable, and SHOULD identify packaging-specific dependencies.
A CMAF or LOC metric MUST NOT be presented as a transport metric when its
observation requires parsing media samples or container boxes.

## Export Formats

An implementation MAY export the registry through OpenMetrics {{OPENMETRICS}},
OpenTelemetry {{OTEL}}, qlog {{MOQ-QLOG}}, or another monitoring system. qlog is
particularly appropriate for event traces and protocol correlation; it is not a
substitute for a low-cardinality time-series registry. OpenMetrics and
OpenTelemetry are appropriate references for metric types, attributes,
temporality, and export, but this document remains responsible for
MoQT-specific scope and semantics.


# Mappings from Existing Vocabularies

## WebRTC Statistics

Many WebRTC-based endpoints can obtain statistics through
`RTCPeerConnection.getStats()`, which returns an `RTCStatsReport`
{{WEBRTC-STATS}}. The presence and quality of a statistic depend on the
endpoint, browser, media pipeline, and codec. A MoQT implementation MUST NOT
claim that a corresponding WebRTC statistic exists when it has not been
collected at the relevant boundary.

The following mappings are informative starting points:

`outbound-rtp.bytesSent`:
: Maps to `moq_media_encoded_bitrate_bps`. Derive a rate over an interval; identify whether bytes are encoded media or include packet overhead.

`outbound-rtp.framesPerSecond`:
: Maps to `moq_media_frame_rate_fps`. Identify the encoded or sent stage.

`outbound-rtp.framesEncoded`:
: Maps to `moq_media_frames_encoded_total`. Media metric; not a MoQT object count.

`outbound-rtp.keyFramesEncoded`:
: Maps to `moq_media_keyframes_encoded_total`. Useful for random-access behavior when present.

`inbound-rtp.bytesReceived`:
: Maps to `moq_media_received_bitrate_bps`. Derive a rate and distinguish media bytes from transport bytes.

`inbound-rtp.jitter`:
: Maps to `moq_media_jitter_seconds`. RTP jitter is not equivalent to MoQT object inter-arrival jitter.

`inbound-rtp.framesDecoded`:
: Maps to `moq_media_frames_decoded_total`. Decoder observation.

`inbound-rtp.framesDropped`:
: Maps to `moq_sub_dropped_frames_total`. Dropped-frame semantics depend on the implementation.

`inbound-rtp.totalDecodeTime`:
: Maps to `moq_media_decode_time_seconds`. Report cumulatively or derive a histogram according to the exporter.

`codec.mimeType`:
: Maps to `moq_media_codec`. Codec metadata, not a numeric metric.

`candidate-pair.currentRoundTripTime`:
: Maps to `moq_link_rtt_seconds`. Candidate-pair RTT is not necessarily the same as MoQT application RTT.

`candidate-pair.availableOutgoingBitrate`:
: Maps to `moq_link_available_bitrate_bps`. An estimate, not an achieved media bitrate.

`candidate-pair.bytesSent`:
: Maps to `moq_link_bytes_sent_total`. Transport-path observation where available.

`candidate-pair.bytesReceived`:
: Maps to `moq_link_bytes_received_total`. Transport-path observation where available.

`transport` or `candidate-pair` state:
: Maps to link or session state attributes. State names MUST retain their source semantics.

Metric names SHOULD NOT imply an exact equivalence between RTP packet metrics
and MoQT object metrics. A media endpoint can export both families when it has
both observations.

## CMCD and CMSD

Common Media Client Data {{CMCD}} describes client-side playback and delivery
context in HTTP adaptive streaming. Common Media Server Data {{CMSD}} describes
server- or intermediary-side information associated with media delivery. The two
vocabularies are complementary: CMCD primarily describes what the client knows
or requests; CMSD primarily describes what a server or intermediary knows or
recommends.

MoQT has no HTTP segment request or response header. Therefore:

* CMCD and CMSD concepts can inform MoQT metric definitions;
* CMCD and CMSD fields MUST NOT be treated as wire-compatible MoQT fields
  without a carriage profile; and
* CMCD and CMSD carriage through HTTP headers or query parameters is outside the
  scope of this document.

The following mapping is informative:

| CMCD/CMSD concept | MoQT analogue | Perspective |
|---|---|---|
| CMCD measured throughput (`mtp`) | `moq_link_available_bitrate_bps` or `moq_media_received_bitrate_bps` | Subscriber; the measurement method MUST be stated. |
| CMCD buffer length (`bl`) | `moq_sub_buffer_level_seconds` | Subscriber media pipeline. |
| CMCD buffer starvation (`bs`) | `moq_sub_buffer_starvation_total` | Subscriber playback. |
| CMCD deadline (`dl`) | `moq_sub_deadline_seconds` | Subscriber application deadline, not a transport timeout. |
| CMCD startup concepts | `moq_sub_startup_delay_seconds` | Subscriber media pipeline. |
| CMCD live-stream latency | `moq_sub_live_latency_seconds` | Requires a clock or timeline definition. |
| CMCD dropped and non-rendered concepts | `moq_sub_dropped_frames_total`, `moq_sub_non_rendered_total` | Subscriber media pipeline. |
| CMSD estimated throughput (`etp`) | `moq_link_available_bitrate_bps` | Server or relay estimate; not the subscriber's achieved rate. |
| CMSD max suggested bitrate (`mb`) | `moq_relay_suggested_bitrate_bps` | A recommendation, not a measured bitrate. |
| CMSD response delay (`rd`) | `moq_relay_request_processing_seconds` | Relay or server processing. |
| CMSD held time (`ht`) | `moq_relay_held_seconds` | Relay or server queue or policy delay. |
| CMSD availability time (`at`) | `moq_relay_object_availability_seconds` | Requires a defined origin and time base. |
| CMSD duress (`du`) | `moq_relay_duress` | Relay operational state. |
{: title="Informative CMCD and CMSD Mapping"}

The mapping intentionally distinguishes measured values, recommendations, and
playback state. For example, a CMSD suggested bitrate is not the same metric as
the subscriber's measured throughput, and CMCD buffer length is not the same as
a relay's object queue.

## qlog and Feedback

The MoQT qlog event definitions {{MOQ-QLOG}} define structured protocol events
such as control message, subgroup, datagram, and FETCH events. These events can
provide trace-level evidence for a metric, but qlog records and time-series
metrics serve different purposes. An implementation MAY correlate them using a
session or trace identifier.

{{MOQFEEDBACK}} defines delivery-quality feedback and a feedback-track
mechanism. This document provides reusable metric names and semantic guidance;
the feedback mechanism defines its own report format, negotiation, routing, and
lifecycle.


# Security Considerations

Metric reports can be used to influence operational decisions such as rate
adaptation, relay selection, and capacity planning. Reports are therefore
subject to spoofing, injection, replay, and resource-exhaustion attacks.

Implementations and deployments SHOULD:

* apply authorization to metric collection separately from media delivery;
* bound label cardinality and report sizes before forwarding or export, so that
  a reporter cannot cause unbounded storage or processing at a collector;
* distinguish measurements from untrusted recommendations or peer assertions;
* protect reports in transit using the security of the selected carriage;
* use end-to-end object protection {{SECURE-OBJECTS}} when relays must not read
  application metrics;
* apply replay, freshness, and sequence handling when reports influence control
  decisions; and
* prevent a reporter from selecting an unauthorized collector or causing a
  node to issue requests to an arbitrary destination.

A metric report MUST NOT be assumed trustworthy merely because it arrived over
an authenticated MoQT session. A subscriber can report incorrect playback state,
and a relay can report incorrect operational state. Consumers SHOULD treat
reports as observations made by a named reporter, not as authoritative truth
about another node.


# Privacy Considerations

Metrics can expose namespace and track names, subscriber activity, network
topology, timing, codec and device information, playback state, and operational
load. They can also be used to fingerprint a user or infer content consumption.

Implementations and deployments SHOULD:

* avoid raw user identifiers and unconstrained object identifiers in labels (see
  {{labels}});
* aggregate subscriber and playback metrics before export where per-subscriber
  detail is not required; and
* document retention and aggregation policies for subscriber and playback
  metrics.


# IANA Considerations

This document has no IANA actions.

A future revision might request a registry for MoQT metric names once the
metric namespace, registration policy, compatibility model, and relationship to
other MoQT metrics work are stable. A carriage document that defines a new track
property, message type, or catalog value is responsible for its own IANA
considerations.


--- back

# Open Issues
{:removeinrfc="true"}

The following questions are open for discussion:

1. Which metrics belong in a small interoperable base registry, and which should
   remain implementation or packaging profiles?
2. Should the registry standardize `moq_` names, semantic identifiers
   independent of export names, or both?
3. Which labels can be safely standardized across deployments without making
   namespace, track, subscriber, or topology privacy assumptions universal?
4. Should structured reports use a common encoding, or should MSF, feedback
   tracks, and external exports define separate encodings over a shared
   registry?
5. Which feedback metrics defined in {{MOQFEEDBACK}} should be referenced by name
   rather than duplicated?
6. Which media and QoE metrics can be defined without requiring a particular
   player, codec, or WebRTC implementation?
7. How should this document align with qlog, OpenMetrics, OpenTelemetry, and
   future relay diagnostics work?

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
