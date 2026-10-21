---

title: BPv7 Echo Service
abbrev: BPEcho
category: std
ipr: trust200902

docname: draft-ietf-dtn-echo-service-latest
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
  latest: https://ricktaylor.github.io/echo-service/draft-ietf-dtn-echo-service.html

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

The echo service is intentionally simple: it does not interpret requests and performs no special processing beyond returning each one to its originator. Defining the response bundle as a wire contract rather than an internal mechanism lets independent implementations interoperate regardless of whether the echo service is built into the Bundle Protocol Agent (BPA) or runs as an ordinary application. A standardized echo service enables diagnostic tools such as ping clients to operate across heterogeneous DTN deployments.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses terminology from the Bundle Protocol Version 7 specification {{!RFC9171}}.

Echo Service:
: A Bundle Protocol service that returns each request to its originator as a response.

Request:
: An application data unit (ADU) delivered ({{!RFC9171, Section 3.1}}) to an echo service at one of its echo service endpoints.

Request Bundle:
: The bundle that carries a request as its payload, addressed to an echo service endpoint.

Response:
: The ADU that an echo service sends in reply to a request: a byte-for-byte copy of that request ({{payload-block}}).

Response Bundle:
: The new bundle, sourced by the echo service node, that carries a response as its payload; {{response-bundle}} defines its content.

Requests and responses are the units of the echo service's behavior; request bundles and response bundles are named where the content or processing of the bundle itself matters. The source or destination of a request or response is that of the bundle that carries it.

# Echo Service Specification

## Service Endpoint

An echo service registers to receive requests at one or more endpoint identifiers. The well-known endpoint is ipn-scheme service number TBD1 ({{iana-considerations}}) on any node; for example, `ipn:2.TBD1` represents the echo service on node number 2. Implementations MAY also support service number 7 for backwards compatibility with existing deployments; service number 7 is in the Private Use range per {{!RFC9758}} and cannot be reserved.

An echo service SHOULD support the well-known endpoint, so that clients can reach it without out-of-band configuration. It MAY also support private or alternative endpoints, which clients must learn of by other means.

## Request Delivery {#request-delivery}

A request bundle is received and processed by the echo service node's BPA like any other bundle destined for one of the node's endpoints: the BPA processes its extension blocks, applies local security policy, reassembles it if it arrived as fragments, and delivers the request to the echo service ({{!RFC9171, Section 5.7}}).

The echo service therefore receives the request, together with delivery metadata from the request bundle ({{!RFC9171, Section 3.1}}): its source node ID and destination EID, and, as far as the BPA makes them available, its bundle processing control flags, lifetime, age, and report-to EID. The echo service uses this metadata only to decide whether to respond ({{when-a-response-is-sent}}) and to set the primary block of the response bundle ({{primary-block}}). It has no other use for the request bundle, and in particular none for its extension blocks.

## The Response Bundle {#response-bundle}

When a request is delivered to an echo service, the echo service responds as specified in this section. When it responds ({{when-a-response-is-sent}}), it MUST request transmission of exactly one response bundle conforming to these rules ({{!RFC9171, Section 5.2}}), and MUST NOT emit more than one response per request.

The response bundle is a new bundle, sourced by the echo service node, whether the echo service runs as an application or is built into the BPA. It is constructed and processed under {{!RFC9171}} (and, where security is applied, {{!RFC9172}}) like any other bundle the node sources, and this document does not restate those rules. The rules in this section constrain only what the echo service supplies for that new bundle: the values of certain primary block fields, which it takes from the delivery metadata of the request ({{request-delivery}}), and its payload, which is the response.

### When a Response Is Sent {#when-a-response-is-sent}

An echo service MUST NOT respond to a request whose source is the null endpoint ID, as no return path exists ({{!RFC9171, Section 4.2.3}}).

An echo service MUST NOT respond to a request that is an administrative record (that is, when the request bundle's "ADU is an administrative record" flag is set), to avoid reflecting status reports and the bundle loops that could result.

An echo service MUST NOT respond to a request whose source is an echo service endpoint: an ipn-scheme EID with the well-known echo service number ({{iana-considerations}}), or an EID that the echo service is configured to recognize as an echo service endpoint (for example, service number 7 where legacy echo services are deployed). Because only an echo service sources bundles from an echo service endpoint, and then only response bundles, such a request is in fact a response, typically elicited by a request whose source was forged; answering it would set two echo services exchanging responses with each other indefinitely.

An echo service MAY also decline to respond to a request as a matter of local policy, for example to enforce access restrictions or rate limits ({{ops-policy}}).

An echo service MUST NOT generate bundles other than response bundles. Other services at the node SHOULD NOT use an echo service endpoint as the source of bundles they originate.

### Primary Block {#primary-block}

The primary block of the response bundle MUST be set as follows. Where these rules take a value from the request bundle, it is from the delivery metadata ({{request-delivery}}); this is the only respect in which the response bundle depends on the request bundle.

Destination:
: MUST be set to the source node ID of the request bundle.

Source:
: If the request bundle was addressed to a singleton endpoint, MUST be set to that echo service endpoint ID; where a node exposes more than one echo service endpoint (for example, both an `ipn` and a `dtn` endpoint), this is the specific endpoint to which the request bundle was addressed. If the request bundle was addressed to a non-singleton endpoint, which cannot serve as the source of a bundle, the source SHOULD be set to the EID of the echo service node's administrative endpoint ({{ops-non-singleton}}).

Creation Timestamp:
: An echo service MUST NOT reuse the request bundle's creation timestamp; the response bundle is assigned its own, as for any bundle the node sources.

Lifetime:
: The response bundle SHOULD be assigned a lifetime that gives it a reasonable opportunity to reach the originator. Like any bundle the node sources, it is subject to the node's local lifetime policy, which bounds the retention cost of response bundles even when a request bundle asserts a very long lifetime. Within that bound, an echo service MAY set the response bundle's lifetime by reusing the request bundle's lifetime value, or by deriving it from the request bundle's observed transit time (its age on reception). A lifetime derived from observed transit time limits retention, but can underestimate the requirement of a return path slower than the forward path, and so benefits from a generous margin.

Report-To:
: If the request bundle requested status reports, an echo service SHOULD set the response bundle's report-to EID to that of the request bundle, so that status reports for both legs of the exchange reach the same observer. Because the request bundle and response bundle are distinct bundles, each report is then attributable to its leg by the source EID of its subject bundle. Otherwise, the report-to EID is set as for any bundle the node sources.

#### Bundle Processing Control Flags {#flags}

The response bundle's bundle processing control flags are set as for any bundle the node sources, except as follows.

"ADU is an administrative record":
: MUST NOT be set. The response is a copy of the request, which is not an administrative record ({{when-a-response-is-sent}}).

"Bundle must not be fragmented":
: SHOULD be copied from the request bundle, unless local policy requires otherwise. A client that sets this flag is typically measuring round-trip time or probing the largest bundle a path will carry ({{size-probing}}), and either measurement is undermined if the response bundle is fragmented on the return path.

"Acknowledgement by application is requested":
: MUST NOT be set. The response does not solicit an application acknowledgement.

Status-report-request flags ("Request reporting of bundle reception", "Request reporting of bundle forwarding", "Request reporting of bundle delivery", and "Request reporting of bundle deletion"):
: MAY be set, typically by mirroring the request bundle's flags, so that an observer can follow both legs of the exchange. These take effect together with the report-to EID (see the Report-To field above).

"Status time requested in reports":
: MAY be set, typically by mirroring the request bundle's value when the echo service mirrors the status-report-request flags, so that reports for the response bundle carry the time of the reported event whenever the requester asked for that detail.

### Payload Block

The response, carried in the payload block of the response bundle, MUST be identical, byte for byte, to the request as delivered to the echo service ({{request-delivery}}). An echo service MUST NOT alter, truncate, reorder, or append to the request in forming the response. Only the request is guaranteed to survive the round trip, which lets a client verify round-trip integrity by comparing the response with the request it sent.

### Extension Blocks {#extension-blocks}

The response bundle carries the extension blocks that the node's normal outbound bundle processing and local security policy add to any bundle it sources — for example a Bundle Age Block, Hop Count Block, Previous Node Block, or BPSec {{!RFC9172}} block — and no others. The extension blocks of the request bundle are processed by the BPA on reception and are not part of what is delivered to the echo service ({{request-delivery}}); none of them is carried into the response bundle.

## Client Considerations

While this specification focuses primarily on the echo service, certain requirements apply to clients to ensure correct operation. This section defines those normative requirements.

### Session Disambiguation {#session-disambiguation}

When multiple clients run concurrently on the same node, each session must be distinguishable so that responses are delivered to the correct client. Multiple concurrent clients on the same node MUST use distinct source endpoint identifiers. Per {{!RFC9171}}, each application instance registers with a unique endpoint ID, and the combination of source and destination provides session disambiguation at the bundle layer without requiring any session identifier in the request.

### Response Authenticity

A response bundle is an independently sourced bundle: its primary block is that of the echo service, and it is not cryptographically bound to the request bundle. A client therefore cannot use a Bundle Integrity Block (BIB) it placed on the request bundle to authenticate the response, because the response bundle is a new bundle and carries no extension block from the request bundle (see {{extension-blocks}}). A client that requires assurance of a response's origin relies instead on the echo service node's own BPSec {{!RFC9172}} policy applied to the response bundle, for example a BIB whose security source is the echo service node. Alternatively, because each response is identical to its request, a client can detect modified or spoofed responses by validating each response against state it retained locally when sending the request; see {{payload-format}}.

### Fragmentation {#fragmentation}

Diagnostic clients SHOULD set the "bundle must not be fragmented" flag in request bundles, unless they intend to exercise fragmentation itself. Fragmentation complicates round-trip time measurement and response verification: fragments might take different paths, arrive out of order, or be lost independently. Setting this flag ensures the request bundle either traverses the network intact or is dropped, providing cleaner diagnostic results. An echo service normally copies the flag to the response bundle ({{flags}}), so the same holds for the return path. Setting the flag also makes it possible to probe the largest bundle a path will carry; see {{size-probing}}.

# Operational Considerations {#ops}

An echo service is a diagnostic tool, and whether a given request should be answered depends on the network in which it operates: the same request can be a legitimate connectivity check in one deployment and reconnaissance or a reflection attempt in another. This document therefore does not enumerate the conditions under which an echo service responds. Instead, it requires that those conditions be under the control of the node operator, and describes the considerations an operator weighs.

## Enabling and Restricting the Echo Service {#ops-policy}

An implementation MUST provide a means for the operator to disable the echo service; to limit the responses it sends, both by restricting which requests it answers and by rate ({{rate-limiting}}); and, where the echo service propagates status-report settings onto response bundles ({{response-bundle}}), to disable that propagation. These means can be provided by the echo service itself, by the BPA through which requests are delivered to it, or by both. A request that the echo service declines to answer is consumed without a response ({{when-a-response-is-sent}}).

Policy can be applied at different points and at different granularities, for example:

- by the BPA before delivery, so that unwanted requests never reach the echo service;
- by source node ID, or by a set of source EIDs (for example, expressed as EID patterns {{?I-D.ietf-dtn-eid-pattern}});
- by the presence on the request bundle of a BPSec {{!RFC9172}} integrity block whose security source the node trusts;
- by the BPA before delivery, according to the extension blocks present on the request bundle, for example discarding request bundles that carry blocks of types the node does not expect;
- by the node from which the request bundle was received, where the convergence layer or a Previous Node Block identifies it; or
- by the echo service endpoint at which the request arrived.

Control over whether the echo service propagates the request bundle's report-to EID and status-report-request flags onto the response bundle matters because propagation lets a requester direct status reports from every node on the return path to an EID of its choosing.

Whether and how an operator applies these limits is a deployment decision. Answering every delivered request maximizes the diagnostic value of a well-known endpoint and simplifies interoperability testing, but exposes the node to the concerns described in {{security-considerations}}. In a network whose bundle sources are not otherwise authenticated, restricting the echo service to requests whose source is authenticated or otherwise trusted is particularly effective against reflection, which relies on forged sources.

## Rate Limiting {#rate-limiting}

In this document, rate limiting means declining to respond to some requests, as described in {{ops-policy}}, rather than delaying responses: a delayed response consumes storage at the echo service node, and to the client it is indistinguishable from a slow return path. An echo service sends no indication to the requester that a response was withheld, because any such indication would itself be a bundle directed at a possibly spoofed source.

The limit, and how it is measured, are matters of local policy. A limit applied per source node ID bounds the traffic the echo service can direct at any one destination; a limit applied per previous node bounds the load any one neighbor can impose. Where the BPA, rather than the echo service, discards request bundles before delivery, it deletes them as specified in {{!RFC9171, Section 5.10}}; any deletion status report it generates (for example, with reason code "Traffic pared") is subject to the node's status-reporting policy and raises the same reflection concern.

## Missing and Multiple Responses {#ops-missing}

Because an echo service sends nothing when it declines a request, a client that receives fewer responses than it sent requests cannot tell, from the responses alone, whether a request or response was lost or expired in transit, whether a request was declined by policy or rate limiting, or whether no echo service is present at all. Loss figures reported by ping clients ({{ping-clients}}) therefore count unanswered requests, whatever the cause, not only path loss. Status reports can narrow the ambiguity where they are available: a delivery report for a request bundle with no corresponding response indicates that the request reached the echo service node, and that the response was then declined or lost on the return path.

Conversely, a client can receive more than one response to a single request, in two different ways:

- Distinct response bundles. Where a request is addressed to a non-singleton endpoint, each member that answers sends its own response in its own response bundle, with its own source and creation timestamp ({{ops-non-singleton}}). These are different bundles, so they are not removed by any duplicate suppression on the return path, and their sources identify the responders.
- Copies of one response bundle. The network can replicate a response bundle on the return path. The copies share its source node ID and creation timestamp, and the client's BPA might or might not suppress them.

Either way, every response is identical to its request, so a client matches responses to requests by content ({{payload-format}}). A ping client typically takes the round-trip time from the first response to arrive, counts the request as answered once, and reports any further responses separately. A client that can see the source node ID and creation timestamp of each response bundle can also tell distinct response bundles apart from copies of the same one.

## Non-Singleton Endpoints {#ops-non-singleton}

Every ipn-scheme EID identifies a singleton endpoint ({{!RFC9758, Section 5.1}}), but other EIDs, such as dtn-scheme EIDs whose demux begins with '~' ({{!RFC9171, Section 4.2.5.1.1}}), can identify endpoints with many members. If an echo service is registered in such an endpoint, a single request elicits a response from every member that receives it. Each echo service still emits at most one response, but the number of responses in the network scales with the membership of the endpoint, reintroducing the count amplification that {{reflection-and-amplification}} otherwise rules out.

An echo service SHOULD NOT answer requests addressed to a non-singleton endpoint unless the operator has explicitly configured it to do so. When it does, the response bundle cannot be sourced from the non-singleton EID to which the request bundle was addressed, because the source of a bundle is a node ID ({{!RFC9171, Section 4.2.5.2}}). The echo service SHOULD instead use the EID of its node's administrative endpoint: for example `ipn:2.0` ({{!RFC9758, Section 5.7}}) or `dtn://node-name/` ({{!RFC9171, Section 4.2.5.1.1}}), preferably in the same URI scheme as the request's destination where the node has an administrative EID in that scheme. Every node is permanently registered in its administrative endpoint ({{!RFC9171, Section 4.2.5.2}}), so this choice needs no additional configuration; where the echo service runs as an application on a BPA that does not let applications source bundles from the administrative endpoint, the BPA can set the source on the echo service's behalf. This choice has three properties that a source shared by the members of the endpoint would lack:

- It contains reflection. Whatever answers a response addresses it to the response's source. Were that source the group's EID, each such answer would be delivered to every member, and two echo groups induced to answer one another, for example by a request to one group forged with the other group's EID as its source, would multiply the traffic by the size of the group at every exchange. A group EID is not normally recognizable as an echo service endpoint, so the rule in {{when-a-response-is-sent}} would not stop this. An administrative endpoint is a single node that does not echo bundles, so any such exchange ends there.
- It keeps response bundles distinct. A bundle is identified by its source node ID and creation timestamp. Members that share a source assign creation timestamps independently and, answering the same request at about the same time, can easily assign the same one; duplicate suppression, fragment reassembly, and status reporting on the return path would then conflate response bundles from different members.
- It identifies the responder, which nothing else in the response bundle does: the response is identical to the request, and a Previous Node Block identifies only the last hop.

A client that addresses a non-singleton endpoint should therefore expect zero, one, or many responses to each request, each sourced from the administrative endpoint of the responding node rather than from the EID the client addressed. Such a client matches responses to requests by content ({{payload-format}}), not by source.

## Endpoint Configuration {#ops-endpoint-config}

The well-known service number requested in {{iana-considerations}} is a default, not a requirement: as {{!RFC9758, Section 5.6}} describes, it lets a client sensibly assume where the service is operating unless configuration says otherwise. Existing deployments answer echo requests on a variety of service numbers. Implementations SHOULD therefore allow the operator to configure the endpoints at which the echo service is registered, and ping clients SHOULD allow the target endpoint to be specified in full.

## Client Source Endpoints

Ping clients need a distinct source endpoint for each concurrent session ({{session-disambiguation}}), usually for the lifetime of that session only. This document does not reserve service numbers for such transient use; a BPA that supports transient registrations can allocate them from a Private Use range of {{!RFC9758}}. Because responses can arrive long after a session has ended, it is good practice for a BPA not to reassign a recently released source endpoint to a different application promptly, and for a client to discard responses that do not match a request it sent.

# Security Considerations

This section discusses security issues relevant to the echo service and potential mitigations.

## Reflection and Amplification

Like any echo or reflection service, an echo service can be abused to direct traffic at a victim. Because the source of a bundle is not authenticated by default, an attacker can send requests whose source is set to a victim's endpoint, causing the echo service to send responses to that victim. The echo service thereby launders the attacker's traffic and conceals its origin.

The volume amplification factor is close to one: each response is identical to its request, and the requirement to emit at most one response per request (see {{response-bundle}}) prevents count amplification. The principal risk is therefore reflection toward, and concentration of traffic on, a chosen victim rather than bandwidth amplification. Operators can mitigate this risk by:

- Rate-limiting responses, particularly to or from previously unseen endpoints ({{rate-limiting}})
- Restricting responses to authenticated or otherwise trusted requesters, for example using Bundle Protocol Security {{!RFC9172}} ({{ops-policy}})
- Monitoring for unusual traffic patterns that might indicate abuse

An echo service that propagates the request bundle's report-to EID and status-report-request flags onto the response bundle (see {{response-bundle}}) extends this reflection surface to the status reports generated for the response bundle; the same mitigations apply, and an operator can also disable that propagation ({{ops-policy}}).

Registering an echo service in a non-singleton endpoint reintroduces count amplification, since every member that receives a request answers it; see {{ops-non-singleton}}.

Because an echo service originates nothing but responses ({{when-a-response-is-sent}}), a bundle whose source is an echo service endpoint is only ever a response bundle. A receiver can use this to recognize reflected traffic: a bundle from an echo service endpoint that does not correspond to a request the destination application sent, for example one delivered to an application that is not a ping client, indicates that the destination's endpoint has been forged as the source of requests elsewhere. An application or BPA MAY discard such bundles, and a BPA MAY apply rate limits or other policy to bundles whose source is a well-known echo service endpoint. Echo services are themselves the most important such receiver, since answering a response would create a loop between two echo services; {{when-a-response-is-sent}} therefore forbids it.

Responses to requests addressed to non-singleton endpoints are sourced from the responding node's administrative endpoint instead ({{ops-non-singleton}}), so an echo service cannot recognize them by their source. This does not reopen the loop: if such a response reaches another echo service, because the request's source was forged as that echo service's endpoint, the second echo service answers it, but its response is addressed to the administrative endpoint, which does not echo bundles, and the exchange ends there.

Source EIDs are not authenticated by default, so these checks help discard reflected traffic but do not authenticate genuine responses; see {{response-authenticity}}.

## Information Disclosure

Echo responses inherently confirm the existence and reachability of the echo service endpoint. Additionally, round-trip time measurements might reveal information about network topology, path characteristics, and store-and-forward delays that could be useful to an adversary mapping a network.

Extension blocks on request bundles can also reveal how nodes on the forward path process bundles, and so which block types they support, how they apply security policy, and potentially which implementations they run ({{block-probing}}). Most of this comes from status reports rather than from the echo service: a block that no node processes, marked "Transmit status report if block can't be processed", elicits a report from every node that receives it where status reporting is enabled, sent to a report-to EID of the requester's choosing. The same is true of a bundle sent to any destination, and the exposure is limited in the same way, by disabling status reporting as {{!RFC9171, Section 8}} recommends by default, or by filtering bundles at the edge of the network. The echo service adds only whether a response returns, and none of the request bundle's extension blocks are carried into the response bundle ({{extension-blocks}}), so the return path is not exposed. Such probing is also a legitimate diagnostic, which this document does not prohibit.

In sensitive environments where this information disclosure is a concern, operators can:

- Restrict echo service access to authenticated endpoints using BPSec, or discard request bundles carrying unexpected extension blocks before delivery ({{ops-policy}})
- Disable the echo service entirely on nodes where diagnostics are not required ({{ops-policy}})
- Deploy the echo service only on designated diagnostic nodes rather than throughout the network

## Resource Exhaustion

An attacker could attempt to exhaust echo service resources by sending a large volume of requests or very large requests. Since the echo service must construct and transmit a response for each request, this creates both memory and bandwidth pressure. The following protections are RECOMMENDED:

- Limiting the maximum size of request accepted
- Limiting the rate of requests answered ({{rate-limiting}})
- Monitoring resource usage, and declining requests when under stress

These protections need not be implemented by the echo service itself. Where the echo service runs as an application on a BPA, the BPA can provide them before requests are delivered, for example by limiting the size of bundles it accepts, the rate at which bundles are delivered to the echo service endpoint, or the storage allotted to such bundles. What matters is that the deployment as a whole provides them, under the operator's control ({{ops-policy}}).

# IANA Considerations

## Well-Known Service Registration

This document requests IANA to register the following entry in the "'ipn' Scheme URI Well-Known Service Numbers for BPv7" registry established by {{!RFC9758}}, assigning the next available value in the Standards Action range (128-255):

| Value | Description  | Reference       |
|-------|--------------|-----------------|
| TBD1  | Echo Service | (this document) |
{: #iana-echo-service align="left" title="Echo Service Registration"}

RFC Editor: please replace TBD1 throughout this document with the value assigned by IANA, and remove this note.

Note: Existing implementations do not agree on a service number for echo. Several use 7 by convention, mirroring the well-known UDP port for the Echo Protocol {{?RFC862}}, while others use different values (for example, 2047). All of these lie within ranges designated Private Use by {{!RFC9758}} and so cannot be reserved. This document requests a value from the Standards Action range to provide a single registered value; see also {{ops-endpoint-config}}.

--- back

# Ping Clients

This appendix provides non-normative guidance for implementing ping clients that use the echo service. While the body of this document defines the echo service behavior, effective ping clients require careful attention to timing, session management, and request design.

## Round-Trip Time Calculation {#rtt-calculation}

Accurate round-trip time (RTT) measurement is the primary purpose of most ping implementations. Ping clients should calculate RTT using locally stored timestamps rather than timestamps embedded in the request:

~~~ pseudocode
RTT = response_receive_time - request_sent_times[sequence_number]
~~~

This approach offers several advantages:

- Requires no clock synchronization between nodes
- Works correctly even if the response is corrupted
- Avoids serialization overhead in the timing path

The client should maintain a map from sequence number to sent timestamp. It should populate the map when each request is transmitted and consult it when each response arrives. Entries should be removed after a configurable timeout to prevent unbounded memory growth.

## Endpoint Selection

As required by {{session-disambiguation}}, multiple concurrent clients on the same node use distinct source endpoint identifiers.

For example, if two concurrent ping sessions on node `ipn:1.0` target `ipn:2.TBD1`, they should use distinct source endpoints such as `ipn:1.1001` and `ipn:1.1002`. The BPA then delivers each response to the correct session based on the destination of its response bundle.

A ping client must not use a well-known echo service endpoint (for example, service number TBD1) as its own source endpoint. Per {{when-a-response-is-sent}}, services other than the echo service do not source bundles from an echo service endpoint, and an echo service does not answer a request so sourced.

## Request Format {#payload-format}

Because each response is identical to its request, the format of requests is entirely at the discretion of the ping client; the echo service neither parses nor interprets them. This document defines no request format.

A client need only place a sequence number in each request, so that it can match each response to the request that produced it. Everything else a client needs is kept in local state, keyed by that sequence number:

- the time the request was sent, used for the round-trip time calculation of {{rtt-calculation}} (timestamps need not be carried in the request); and
- optionally, whatever lets the client confirm that a response is an unmodified echo of its own request — for example, by retaining each request it sent and comparing it against the response.

Alongside the sequence number, a client should include a value that it does not reuse across sessions and that an off-path attacker cannot predict, such as a random nonce. This lets the client reject responses belonging to an earlier session that used the same source endpoint, and responses forged by an attacker able to guess sequence numbers; see also {{ops-missing}}.

Because the whole request returns unchanged, a client can match and validate responses with no agreed wire format and without relying on Bundle Protocol Security in the network. Authenticating the responder itself is a separate matter, addressed by the echo service node's own BPSec policy on the response bundle ({{response-authenticity}}).

A client may also pad the request with additional bytes to reach a chosen request bundle size, for example to probe the largest bundle a path will carry ({{size-probing}}).

## Interpreting Status Reports {#status-reports}

A client may request status reports on its request bundles in the usual way, by setting the relevant bundle processing control flags and a report-to EID. Such reports concern the request bundle's own progress toward the echo service and carry the client's endpoint as the source EID of the subject bundle.

By setting status-report-request flags and a report-to EID on its request bundle, a ping client asks to observe the exchange end-to-end. An echo service can honor this by propagating those settings onto the response bundle (see {{response-bundle}}). Where it does, the designated observer receives status reports for both legs of the exchange and can attribute each report to its leg from the source EID of the report's subject bundle: the client's endpoint for the request bundle, and for the response bundle the echo service endpoint or, where the request was addressed to a non-singleton endpoint, the responding node's administrative endpoint (see {{ops-non-singleton}}). Because not every echo service does so, and status reporting is optional in any case, a client should not rely on receiving reports for the return leg. Note also that a client can tie a request-leg report to a specific request, because it knows that bundle's identifier, but cannot in general tie a response-leg report to a specific request: the response bundle's creation timestamp is assigned by the echo service and is not predictable by the client. Round-trip completion for a given request is instead confirmed by the response, which carries the sequence number (see {{rtt-calculation}}).

Status reports on the request bundle can still aid diagnosis. For example, a "Forwarded" report with no corresponding response suggests the request bundle was lost between an intermediate node and the echo service. Note that status reporting is optional per {{!RFC9171}} and many BPA implementations disable it by default; clients should not rely on receiving status reports for correct operation.

## Path Probing {#path-probing}

A response confirms more than reachability: it shows that the request bundle, as the client constructed it, was carried to the echo service node and that a response bundle came back. By varying how it constructs request bundles, a ping client can use this to test how a path treats particular bundles before an application relies on them. This section describes two such probes: varying the extension blocks on request bundles, and varying their size.

### Extension Block Probing {#block-probing}

A request bundle is an ordinary bundle sourced by the client's node, so the client can attach extension blocks to it, and set their block processing control flags ({{!RFC9171, Section 4.2.4}}), as for any bundle it sources. A response then shows that a request bundle carrying those blocks was received, processed, and forwarded by every node on the forward path, and delivered at the echo service node. A client can use this to test, for example, whether the nodes on a path support a new extension block type, or accept a BPSec {{!RFC9172}} block with a particular security context.

The block attached can be of a type whose support is under test, or of a type that no node on the path is expected to process, such as one from the private and experimental range ({{!RFC9171, Section 4.3.2}}), in which case it serves purely as a signal. Either way, the client sets "Transmit status report if block can't be processed" on the block and a report-to EID that it monitors. Each node that receives the request bundle and cannot process the block then generates a reception status report with reason code "Block unsupported", where status reporting is enabled ({{!RFC9171, Section 5.6}}); nodes that can process the block do not. The block's other flags determine what happens next:

- With neither "Delete bundle if block can't be processed" nor "Discard block if it can't be processed" set, the request bundle continues with the block intact. Every node on the forward path that cannot process the block reports, so a block that no node processes elicits a report from each node in turn, and the response still confirms the path end to end.
- With "Discard block if it can't be processed" set, the first node that cannot process the block reports and removes it, and the request bundle continues without it. The report identifies where support for the block ends, and the response is unaffected.
- With "Delete bundle if block can't be processed" set, the first node that cannot process the block reports and deletes the request bundle, so no response returns. A response then shows that every node on the forward path, including the echo service node, processed the block, even where status reporting is disabled.

A Hop Count Block ({{!RFC9171, Section 4.4.3}}) can be used in a similar way to bound how far a request bundle travels: a request bundle whose hop count exceeds its hop limit is deleted for the reason "Hop limit exceeded", and a deletion status report, where requested and generated, identifies the node at which this happened.

Such probes test only the forward path. The response bundle is a new bundle that carries none of the request bundle's extension blocks ({{extension-blocks}}), so the return path is not tested; testing it needs a request sent in the opposite direction. A request can also go unanswered for reasons unrelated to the block under test ({{ops-missing}}), including policy at the echo service node that discards request bundles carrying unexpected extension blocks ({{ops-policy}}). A missing response is therefore best interpreted against a control: a request bundle without the block under test, sent along the same path at about the same time. A response to the control but not to the probe points to the block; no response to either says nothing about it. Status reports add detail where they are available, but are often disabled ({{status-reports}}).

Attaching extension blocks to a request bundle depends on the interface that the client's BPA offers to applications. A client that can only submit ADUs for transmission might be unable to do so, so this kind of probing is better suited to diagnostic tools with lower-level access to their BPA.

### Bundle Size Probing {#size-probing}

A client can probe the largest bundle that a path will carry without fragmentation by sending request bundles of increasing size, padding the request ({{payload-format}}), with "Bundle must not be fragmented" set ({{fragmentation}}). Because a response is identical to its request, the response bundle carries the same data on the return path, so unlike extension block probes, a size probe tests both directions: a response shows that the forward path carried the request bundle, and the return path the response bundle, each unfragmented.

This resembles packetization layer path MTU discovery {{?RFC8899}} rather than ICMP-based path MTU discovery. {{!RFC9171}} defines no status report reason code indicating that a bundle was too large to forward unfragmented; a node that cannot forward such a bundle might retain it until a sufficiently long contact, or delete it for another reason, such as "No timely contact with next node on route". A client therefore learns only from success, and a result is subject to the following limits:

- The limit is not a fixed property of the path. It depends on convergence-layer limits, such as the TCPCL Transfer MRU {{?RFC9174}}, and on the volume of upcoming contacts and the storage available at each node ({{!RFC9171, Section 5.8}}), and so can vary over time.
- The client controls only the size of the request bundle as it leaves its node. Nodes on the path can add extension blocks, and the response bundle carries the echo service node's own primary block and extension blocks ({{extension-blocks}}), so it can be larger or smaller than the request bundle. A result is approximate by the size of those blocks.
- An echo service node can limit the size of request it accepts ({{resource-exhaustion}}), which caps the size that can be probed, and a missing response cannot by itself be distinguished from loss or rate limiting ({{ops-missing}}). Interleaving small control requests helps to separate a size limit from other causes, and a delivery status report for a large request bundle, where available, shows that the forward path carried it and so places the limit on the return path.

## Statistics

Ping clients should track and report standard statistics consistent with traditional IP ping:

- Requests transmitted
- Responses received
- Loss percentage
- RTT minimum, average, maximum, and standard deviation

These statistics provide a quick assessment of network health and help identify routing problems, congestion, or intermittent connectivity.

Example output format following ICMP ping conventions:

~~~ text
--- ipn:2.TBD1 ping statistics ---
5 requests transmitted, 4 received, 20% loss
rtt min/avg/max/stddev = 1.234/2.567/4.891/1.203 s
~~~
