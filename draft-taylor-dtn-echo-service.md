---

title: BPv7 Echo Service
abbrev: BPEcho
category: std
ipr: trust200902

docname: draft-taylor-dtn-echo-service-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: Internet
workgroup: Delay/Disruption Tolerant Networking
keyword:

- DTN
- BPv7
- Bundle Protocol
- Echo
venue:
  group: Delay/Disruption Tolerant Networking
  type: Working Group
  mail: dtn@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dtn/
  github: ricktaylor/echo-service
  latest: https://ricktaylor.github.io/echo-service/draft-taylor-dtn-echo-service.html

author:

- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: rtaylor@aalyria.com

normative:

informative:

--- abstract

This document specifies an echo service for Bundle Protocol Version 7 (BPv7) networks. An echo service receives bundles at a well-known endpoint and replies to each with a response bundle that returns the payload to the originator. This enables round-trip time measurement and end-to-end connectivity verification in Delay-Tolerant Networks. This document requests IANA allocation of a well-known IPN service number for the echo service.

--- middle

# Introduction

Delay-Tolerant Networks (DTNs) present unique challenges for network diagnostics. Unlike traditional IP networks where ICMP Echo (ping) provides immediate feedback, DTN bundles might traverse store-and-forward paths with significant delays. Nevertheless, the ability to verify end-to-end connectivity and measure round-trip time remains essential for network operators.

This document specifies an echo service for Bundle Protocol Version 7 {{!RFC9171}}. An echo service receives bundles at a well-known endpoint and replies to each with a single response bundle that returns the payload to the originator.

This document specifies the externally observable content of the response bundle as a set of conformance rules; it does not constrain how an implementation produces that bundle. An implementation may construct a new bundle, or may derive the response from the request internally, provided the response conforms to the rules in {{response-bundle}}. Because the response is an ordinary bundle, the node's Bundle Protocol Agent (BPA) handles its routing, extension-block processing, and forwarding as it would for any other bundle the node sources.

The echo service is intentionally simple: it interprets no payload and performs no special processing beyond returning the payload to its originator. Defining the response as a wire contract rather than an internal mechanism lets independent implementations interoperate regardless of whether the echo service is built into the BPA or runs as an ordinary application. A standardized echo service enables diagnostic tools such as ping clients to operate across heterogeneous DTN deployments.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses terminology from the Bundle Protocol Version 7 specification {{!RFC9171}}.

Echo Service:
: A Bundle Protocol service that returns the payload of each received bundle to its originator in a response bundle.

Request Bundle:
: A bundle received by an echo service at one of its echo service endpoints.

Response Bundle:
: The bundle an echo service sends in reply to a request bundle; {{response-bundle}} defines its content.

# Echo Service Specification

## Service Endpoint

An echo service registers to receive bundles at one or more endpoint identifiers. The well-known endpoint is IPN-scheme service number 128 on any node; for example, `ipn:2.128` represents the echo service on node number 2. Implementations MAY also support service number 7 for backwards compatibility with existing deployments; service number 7 is in the Private Use range per {{!RFC9758}} and cannot be reserved.

An echo service SHOULD support the well-known endpoint, so that clients can reach it without out-of-band configuration. It MAY also support private or alternative endpoints, which clients must learn of by other means.

## The Response Bundle {#response-bundle}

Upon receiving a bundle at one of its echo service endpoints (the "request bundle"), an echo service generates a response as specified in this section. When a response is sent ({{when-a-response-is-sent}}), the echo service MUST submit to its BPA, for transmission, exactly one bundle (the "response bundle") conforming to these rules, and MUST NOT emit more than one response bundle per request bundle.

These rules constrain only the content of the response bundle, not the mechanism by which it is produced: an implementation MAY construct a new bundle or MAY derive the response from the request internally, provided the emitted response bundle conforms. Aside from the constraints in this section, the response bundle is an ordinary bundle, constructed and processed under {{!RFC9171}} (and, where security is applied, {{!RFC9172}}) like any other bundle the node sources; this document does not restate those rules.

### When a Response Is Sent {#when-a-response-is-sent}

An echo service MUST NOT generate a response bundle when the source node ID of the request bundle is the null endpoint ID, as no return path exists ({{!RFC9171, Section 4.2.3}}).

An echo service MUST NOT generate a response bundle when the request bundle's payload is an administrative record (the "ADU is an administrative record" flag is set), to avoid reflecting status reports and the bundle loops that could result.

An echo service MUST NOT generate bundles other than response bundles. Other services at the node SHOULD NOT use an echo service endpoint as the source of bundles they originate.

### Primary Block

The primary block of the response bundle MUST be set as follows.

Destination:
: MUST be set to the source node ID of the request bundle.

Source:
: MUST be set to the echo service endpoint ID at which the request bundle was received. Where a node exposes more than one such endpoint (for example, both an `ipn` and a `dtn` endpoint), the source MUST be the specific endpoint to which the request bundle was addressed.

Creation Timestamp:
: An echo service MUST NOT reuse the request bundle's creation timestamp; the response bundle is assigned its own, as for any bundle the node sources.

Lifetime:
: The response bundle SHOULD be assigned a lifetime that gives it a reasonable opportunity to reach the originator. Like any bundle the node sources, it is subject to the node's local lifetime policy, which bounds the retention cost of responses even when a request asserts a very long lifetime. Within that bound, an echo service MAY set the response lifetime by reusing the request bundle's lifetime value, or by deriving it from the request bundle's observed transit time (its age on reception). A lifetime derived from observed transit time limits retention, but can underestimate the requirement of a return path slower than the forward path, and so benefits from a generous margin.

Report-To:
: If the request bundle requested status reports, an echo service SHOULD set the response bundle's report-to EID to that of the request bundle, so that status reports for both legs of the exchange reach the same observer. Because the request and response are distinct bundles, each report is then attributable to its leg by the source EID of its subject bundle. Otherwise, the report-to EID is set as for any bundle the node sources.

### Bundle Processing Control Flags {#flags}

An echo service does not copy the request bundle's control flags wholesale. The following per-flag requirements specify the response bundle's control flags; any flag not listed must not just be copied verbatim from the request bundle, but is set as for any bundle the node sources.

"ADU is an administrative record":
: MUST NOT be set. The response carries the reflected application payload, which is not an administrative record.

"Bundle must not be fragmented":
: MAY be copied from the request bundle.

"Acknowledgement by application is requested":
: MUST NOT be set. The response does not solicit an application acknowledgement.

Status-report-request flags ("Request reporting of bundle reception", "Request reporting of bundle forwarding", "Request reporting of bundle delivery", and "Request reporting of bundle deletion"):
: MAY be set, typically by mirroring the request bundle's flags, so that an observer can follow both legs of the exchange. These take effect together with the report-to EID (see the Report-To field above).

"Status time requested in reports":
: MAY be set, typically by mirroring the request bundle's value when the echo service mirrors the status-report-request flags, so that reports for the response carry the time of the reported event whenever the requester asked for that detail.

### Payload Block

The response bundle MUST contain a payload block whose content is identical, byte for byte, to the payload block content of the request bundle. An echo service MUST NOT alter, truncate, reorder, or append to the payload. This reflected payload is the only content guaranteed to survive the round trip; it allows a client to verify round-trip integrity by comparing the returned payload with what it sent.

### Extension Blocks {#extension-blocks}

An echo service is not obliged to reproduce any extension block from the request bundle in the response bundle.

An echo service MAY include extension blocks in the response bundle — for example a Bundle Age Block, Hop Count Block, Previous Node Block, or BPSec {{!RFC9172}} block — as determined by the normal outbound bundle processing and local security policy of its node.

An echo service that copies an extension block from the request bundle into the response remains bound by that block's own specification. In particular, it cannot assume the security role of a BPSec block's original security source, so copying BPSec blocks from the request is generally not meaningful.

## Client Considerations

While this specification focuses primarily on the echo service, certain requirements apply to clients to ensure correct operation. This section defines those normative requirements.

### Session Disambiguation {#session-disambiguation}

When multiple clients run concurrently on the same node, each session must be distinguishable so that responses are delivered to the correct client. Multiple concurrent clients on the same node MUST use distinct source endpoint identifiers. Per {{!RFC9171}}, each application instance registers with a unique endpoint ID, and the combination of source and destination provides session disambiguation at the bundle layer without requiring any session identifier in the payload.

### Response Authenticity

A response bundle is an independently sourced bundle: its primary block is that of the echo service, and it is not cryptographically bound to the request bundle. A client therefore cannot use a Bundle Integrity Block (BIB) it placed on the request to authenticate the response, because the request's BIB is not carried into the response (see {{extension-blocks}}). A client that requires assurance of a response's origin relies instead on the echo service node's own BPSec {{!RFC9172}} policy applied to the response bundle, for example a BIB whose security source is the echo service node. Alternatively, because the echo service returns the payload unchanged, a client can detect modified or spoofed responses by validating the returned payload against state it retained locally when sending the request; see {{payload-format}}.

### Fragmentation {#fragmentation}

Diagnostic clients MAY set the "bundle must not be fragmented" flag in bundles sent to the echo service. Fragmentation complicates round-trip time measurement and payload verification: fragments might take different paths, arrive out of order, or be lost independently. Setting this flag ensures the bundle either traverses the network intact or is dropped, providing cleaner diagnostic results.

If a client needs to test path MTU, it can send bundles of increasing size with fragmentation disabled and observe which sizes succeed. This approach directly reveals the path's maximum bundle size rather than relying on fragmentation behavior.

# Security Considerations

This section discusses security issues relevant to the echo service and potential mitigations.

## Reflection and Amplification

Like any echo or reflection service, an echo service can be abused to direct traffic at a victim. Because the source of a bundle is not authenticated by default, an attacker can send request bundles whose source is set to a victim's endpoint, causing the echo service to send response bundles to that victim. The echo service thereby launders the attacker's traffic and conceals its origin.

The volume amplification factor is close to one: an echo service returns the payload unchanged, and the requirement to emit at most one response bundle per request bundle (see {{response-bundle}}) prevents count amplification. The principal risk is therefore reflection toward, and concentration of traffic on, a chosen victim rather than bandwidth amplification. To mitigate this risk, implementations SHOULD:

- Rate-limit response bundles, particularly to or from previously unseen endpoints
- Monitor for unusual traffic patterns that might indicate abuse
- Consider requiring authentication via Bundle Protocol Security {{!RFC9172}} in sensitive deployments

An echo service that propagates the request bundle's report-to EID and status report requests onto the response (see {{response-bundle}}) extends this reflection surface to the status reports generated for the response; the same mitigations apply.

## Information Disclosure

Echo responses inherently confirm the existence and reachability of the echo service endpoint. Additionally, round-trip time measurements might reveal information about network topology, path characteristics, and store-and-forward delays that could be useful to an adversary mapping a network.

In sensitive environments where this information disclosure is a concern, operators MAY:

- Restrict echo service access to authenticated endpoints using BPSec
- Disable the echo service entirely on nodes where diagnostics are not required
- Deploy the echo service only on designated diagnostic nodes rather than throughout the network

## Resource Exhaustion

An attacker could attempt to exhaust echo service resources by sending a large volume of bundles or bundles with very large payloads. Since the echo service must construct and transmit a response bundle for each request, this creates both memory and bandwidth pressure. Implementations SHOULD:

- Limit the maximum payload size accepted for reflection
- Implement rate limiting on both connections and bundle processing
- Monitor resource usage and reject or delay bundle processing when under stress

# IANA Considerations

## Well-Known Service Registration

This document requests IANA to register the following entry in the "'ipn' Scheme URI Well-Known Service Numbers for BPv7" registry established by {{!RFC9758}}:

| Value | Description  | Reference       |
|-------|--------------|-----------------|
| 128   | Echo Service | (this document) |
{: #iana-echo-service align="left" title="Echo Service Registration"}

For the IPN scheme, the service number is appended to the node number; for example, `ipn:2.128` is the echo service on node number 2.

Note: Existing implementations do not agree on a service number for echo. Several use 7 by convention, mirroring the well-known UDP port for the Echo Protocol {{?RFC862}}, while others use different values (for example, 2047). All of these lie within ranges designated Private Use by {{!RFC9758}} and so cannot be reserved. This document requests service number 128, the lowest value in the Standards Action range, to provide a single registered value. Implementations MAY continue to support service number 7, or other values, for backwards compatibility.

--- back

# Ping Clients

This appendix provides non-normative guidance for implementing ping clients that use the echo service. While the body of this document defines the echo service behaviour, effective ping clients require careful attention to timing, session management, and payload design.

## Round-Trip Time Calculation {#rtt-calculation}

Accurate round-trip time (RTT) measurement is the primary purpose of most ping implementations. Ping clients should calculate RTT using locally stored timestamps rather than timestamps embedded in the payload:

~~~ pseudocode
RTT = response_receive_time - request_sent_times[sequence_number]
~~~

This approach offers several advantages:

- Requires no clock synchronization between nodes
- Works correctly even if the payload is corrupted
- Avoids serialization overhead in the timing path

The client should maintain a map from sequence number to sent timestamp. It should populate the map when each request is transmitted and consult it when each response arrives. Entries should be removed after a configurable timeout to prevent unbounded memory growth.

## Endpoint Selection

As required by {{session-disambiguation}}, multiple concurrent clients on the same node use distinct source endpoint identifiers.

For example, if two concurrent ping sessions on node `ipn:1.0` target `ipn:2.128`, they should use distinct source endpoints such as `ipn:1.1001` and `ipn:1.1002`. The bundle protocol agent will then route responses back to the correct session based on the destination of the response bundle.

A ping client must not use a well-known echo service endpoint (for example, service number 128) as its own source endpoint. Per {{when-a-response-is-sent}}, services other than the echo service do not source bundles from an echo service endpoint; a request so sourced would direct the response to the local echo service rather than to the client.

## Payload Format {#payload-format}

The echo service returns the payload unchanged, so its format is entirely at the discretion of the ping client; the service neither parses nor interprets it. This document defines no payload format.

A client need only place a sequence number in the payload, so that it can match each response to the request that produced it. Everything else a client needs is kept in local state, keyed by that sequence number:

- the time the request was sent, used for the round-trip time calculation of {{rtt-calculation}} (timestamps need not be carried in the payload); and
- optionally, whatever lets the client confirm that a response is an unmodified echo of its own request — for example, by retaining the payload it sent and comparing it against the payload returned.

Because the whole payload returns unchanged, a client can match and validate responses with no agreed wire format and without relying on Bundle Protocol Security in the network. Authenticating the responder itself is a separate matter, addressed by the echo service node's own BPSec policy on the response.

A client may also pad the payload with additional bytes to reach a chosen total bundle size. Because the echo service reflects the payload unchanged, padding the request is a simple and effective way to probe the largest bundle a path will carry; see {{fragmentation}}.

## Interpreting Status Reports {#status-reports}

A client may request status reports on its request bundles in the usual way, by setting the relevant bundle processing control flags and a report-to EID. Such reports concern the request bundle's own progress toward the echo service and carry the client's endpoint as the source EID of the subject bundle.

By setting status-report-request flags and a report-to EID on its request, a ping client asks to observe the exchange end-to-end. An echo service can honour this by propagating those settings onto the response bundle (see {{response-bundle}}). Where it does, the designated observer receives status reports for both legs of the exchange and can attribute each report to its leg from the source EID of the report's subject bundle: the client's endpoint for the request, and the echo service endpoint for the response. Because not every echo service does so, and status reporting is optional in any case, a client should not rely on receiving reports for the return leg. Note also that a client can tie a request-leg report to a specific request, because it knows that bundle's identifier, but cannot in general tie a response-leg report to a specific request: the response's creation timestamp is assigned by the echo service and is not predictable by the client. Round-trip completion for a given request is instead confirmed by the reflected payload, which carries the sequence number (see {{rtt-calculation}}).

Status reports on the request bundle can still aid diagnosis. For example, a "Forwarded" report with no echo response suggests the request was lost between an intermediate node and the echo service. Note that status reporting is optional per {{!RFC9171}} and many BPA implementations disable it by default; clients should not rely on receiving status reports for correct operation.

## Statistics

Ping clients should track and report standard statistics consistent with traditional IP ping:

- Bundles transmitted
- Responses received
- Packet loss percentage
- RTT minimum, average, maximum, and standard deviation

These statistics provide a quick assessment of network health and help identify routing problems, congestion, or intermittent connectivity.

Example output format following ICMP ping conventions:

~~~ text
--- ipn:2.128 ping statistics ---
5 bundles transmitted, 4 received, 20% loss
rtt min/avg/max/stddev = 1.234/2.567/4.891/1.203 s
~~~
