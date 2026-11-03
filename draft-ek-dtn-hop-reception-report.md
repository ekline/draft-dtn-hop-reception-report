---
title: "Bundle Protocol Hop Reception Reports"
abbrev: "BP Hop Reception Report"
category: std

docname: draft-ek-dtn-hop-reception-report-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Delay/Disruption Tolerant Networking"
keyword:
 - DTN
 - BPv7
 - reception report
 - extension block
 - administrative record
venue:
  group: "Delay/Disruption Tolerant Networking"
  type: "Working Group"
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ekline/draft-dtn-hop-reception-report"
  latest: "https://ekline.github.io/draft-dtn-hop-reception-report/draft-ek-dtn-hop-reception-report.html"

author:
 -
    fullname: "Erik Kline"
    organization: Aalyria Technologies
    email: "ek.ietf@gmail.com"

normative:

informative:

...

--- abstract

This document defines the Hop Reception Report Request block for Bundle
Protocol version 7 (BPv7). A sending node uses this extension block to
request a reception report from the next-hop node. The bundle source can
also request a report on initial transmission.

The report goes to the sender's administrative endpoint. The receiver
validates this endpoint from the Previous Node block. If that block is
absent, it uses convergence-layer peer authentication or trusted
configuration to establish the sender's identity and administrative
endpoint.

This document also defines the Hop Reception Report administrative record.
It batches positive reception assertions for one or more bundles or
fragments for the same requesting neighbor. These reports do not transfer
custody or responsibility for the reported bundles.

--- middle

# Introduction {#intro}

A Bundle Protocol version 7 (BPv7) {{!RFC9171}} node uses a Convergence
Layer Adapter (CLA) to send a bundle to a neighbor. The CLA can usually
indicate that the transfer completed. The guarantees of that indication
depend on the convergence-layer specification and implementation.
Completion does not necessarily establish that the peer's Bundle Protocol
Agent (BPA) received the bundle. If completion indicates only
transport-level receipt, a failure before BPA reception can still cause
bundle loss.

BPv7 Bundle Status Reports ({{!RFC9171, Section 6.1.1}}) report bundle
reception, forwarding, delivery, or deletion. Only the bundle source can
request these reports and select their destination. It sets these values
in the primary block, which cannot change after creation. A source can
request reception reports from nodes along the path, subject to their
reporting policies. A forwarding node cannot change these primary-block
flags to request a report for itself.

The Hop Reception Report Request block lets a sending node request a
reception report from its next hop. Both transit-forwarding nodes and
bundle sources can use it. The request block contains no address.

If a Previous Node block ({{!RFC9171, Section 4.4.1}}) is present, the
requesting node puts its own administrative endpoint ID in that block.
The ID must be consistent with the established sending-peer identity. If
the block is absent, the receiver can use convergence-layer peer
authentication or trusted configuration. That association must establish
the immediate sender's identity and administrative endpoint for the
subject reception ({{previous-node}}).

The response uses a Hop Reception Report administrative record with a
nonempty batch of positive reception assertions. A request block is
required in both cases. Sender identification alone does not trigger
reporting.

A Hop Reception Report has the same reception meaning as the "reporting
node received bundle" status assertion in {{!RFC9171, Section 6.1.1}}.
The BPA generates that assertion at Step 2 of {{!RFC9171, Section 5.6}},
before subsequent validation.

A Hop Reception Report asserts only that the reporting BPA received the
identified bundle or fragment. It does not assert that the bundle passed
validation, security verification, or ingress-policy checks. It does not
assert acceptance for forwarding or delivery. The receiving BPA can later
reject or delete the bundle without contradicting the reception assertion.
The report does not establish completion of all reception processing,
durable storage, or continued retention. It implies
no commitment to retain, forward, or deliver the bundle and does not
transfer custody or responsibility for the bundle between nodes.

Hop Reception Reports provide explicit BPA reception assertions,
independent of the convergence layer. Local policy determines how a
requesting BPA uses this signal. That policy is outside this document's
scope.

"One hop" describes the reception event and the requesting neighbor.
Nodes that support this specification remove the request block before
onward forwarding. The required block processing control flags cause
other nodes to discard it. Addressing the report to the requesting
neighbor does not restrict its route to the immediate link. Local policy
determines the return route.

## Open Issues {#open-issues}

RFC EDITOR: Please remove this section before publication.

The following point is noted for working group discussion:

Reception timestamp:
: This revision includes a reception time in an entry if and only if the
  subject bundle's "Report status time" flag is set. A future revision
  could allow the requesting node to request timestamps independently of
  that source-controlled flag. No such request flag is defined here.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

BPA:
: Bundle Protocol Agent, as defined in {{!RFC9171}}.

CLA:
: Convergence Layer Adapter, as defined in {{!RFC9171}}.

Requesting node:
: The immediate sending BPA that inserts a Hop Reception Report Request
  block before sending the subject bundle to a neighbor. This can be a
  transit-forwarding node or the bundle source on initial transmission.

Receiving node:
: The node that receives the subject bundle from the requesting node and
  processes the Hop Reception Report Request block.

Subject bundle:
: The bundle or fragment whose reception is reported.

Carrier bundle:
: The bundle whose payload contains a Hop Reception Report administrative
  record.

EID:
: Endpoint ID, as defined in {{!RFC9171}}.

## EID Comparison {#eid-comparison}

Nodes MUST compare EIDs by their decoded values, using the applicable
URI-scheme matching rules, not by their CBOR encoding bytes. This rule
applies to:

- Subject-source EIDs in bundle identifiers.
- Requester EIDs used to group entries into batches.
- Reporting-node and peer-identity correlation.

Equivalent two-element and three-element `ipn` scheme-specific encodings
MUST match
as specified in {{!RFC9758, Section 6.4}}. This comparison rule does not
alter security-context rules for integrity verification or the wire
format of any identifier.

Distinct EIDs MUST NOT be treated as equivalent merely because they
appear to identify services on the same node. Authentication or
trusted configuration MUST establish any association between different
node identities used for peer or reporter correlation. That association
does not make distinct subject-source EIDs or report destinations
interchangeable.

# Hop Reception Report Request Block {#block}

## Block Definition

The Hop Reception Report Request block is a BPv7 canonical block with a
block type code of TBD1 (see {{iana}}).

The block-type-specific data SHALL be a single CBOR unsigned integer
{{!RFC8949}} that represents a flags field. This document defines no flags.
The sender SHALL set the value to zero (0). Receiving nodes SHALL ignore
flag bits they do not recognize.

The block SHALL NOT appear more than once in a bundle.

## Identification of the Requesting Node {#previous-node}

A Hop Reception Report Request block MUST be present to trigger a Hop
Reception Report. Before honoring the request, the receiver MUST establish
the immediate sending BPA's identity and administrative endpoint for the
subject bundle. The following cases apply:

- When a Previous Node block ({{!RFC9171, Section 4.4.1}}) is present, its
  node ID MUST identify the immediate sending BPA and MUST be consistent
  with the peer identity established through convergence-layer
  authentication or trusted configuration. A requesting node using a
  Previous Node block MUST place its own administrative endpoint ID in
  that block. The report destination is that supplied, validated endpoint.
- When no Previous Node block is present, the receiver MAY honor the
  request if it can establish both the sending BPA's identity and
  administrative endpoint. It uses convergence-layer peer authentication
  or trusted configuration for this association. The report destination
  is the administrative endpoint established by that association.

In both cases, the association MUST apply to the subject bundle's
reception. A transport address or an unauthenticated
advertised node ID alone is insufficient. If the required association
cannot be established, the receiver MUST NOT generate a Hop Reception
Report for that request. A malformed, duplicated, or inconsistent Previous
Node block MUST NOT be treated as absent to enable fallback. The receiver
MUST NOT honor a request if a present Previous Node block conflicts with
the established sending-peer identity.

The receiver MUST NOT infer the immediate sender solely from the primary
block's source node ID. That field can identify a source several hops
away. Although {{!RFC9171, Section 4.2.5.2}} permits other singleton
endpoint IDs to identify a node, this mechanism requires its administrative
endpoint so the report reaches the administrative element. For `ipn`,
administrative endpoints use Service Number zero as specified in
{{!RFC9758, Section 5.7}}. Endpoint syntax alone proves neither peer
identity nor authorization. A receiver MUST NOT substitute a destination
by converting an arbitrary node ID without the required authenticated or
configured association. EID comparison follows {{eid-comparison}}.

This procedure adds no address field to the request block. Local policy
still determines whether an otherwise valid request is honored
({{receiving}}).

## Block Processing Control Flags {#flags}

A requesting node inserting the Hop Reception Report Request block SHALL
set the block processing control flags ({{!RFC9171, Section 4.2.4}}) as
follows. These flags are distinct from the zero-valued flags field in the
block-type-specific data.

- "Block must be replicated in every fragment" (bit 0): SHOULD be set to
  1, so that a node receiving only a fragment can report reception of that
  fragment. Hop Reception Report entries carry fragment offset and payload
  length, so distinct fragments are distinguishable.
- "Transmit status report if block can't be processed" (bit 1): MUST be 0.
- "Delete bundle if block can't be processed" (bit 2): MUST be 0. Inability
  to honor a conforming request does not require deletion of its bundle.
- "Discard block if it can't be processed" (bit 4): MUST be 1, so that a
  node that does not implement this specification removes the block
  before forwarding the bundle further, preserving the one-hop scope.

# Hop Reception Report Administrative Record {#record}

The Hop Reception Report is a BPv7 administrative record with record type
code TBD2 (see {{iana}}), distinct from ordinary Bundle Status Report type
1. It SHALL use the two-element CBOR administrative-record envelope of
{{!RFC9171, Section 6.1}}, as updated by {{!RFC9713}}:

~~~
[TBD2, [reception-entry, ...]]
~~~

The record content SHALL be an array containing at least one reception
entry. A batch MAY contain any positive number of entries within
applicable bundle-size and implementation limits. This specification sets
no fixed protocol maximum and does not require unlimited resources. Each
entry positively asserts reception of the identified bundle or fragment.
Omission of an identifier conveys no negative assertion. The record has no status indicators, ordinary Bundle Status
Report status arrays, or reason codes. It conveys no custody, refusal,
forwarding, delivery, or deletion status.

Each entry SHALL be an array containing a bundle identifier and, when
required below, a reception time. For a non-fragment, the identifier SHALL
be a two-item array: source node ID, then creation timestamp. For a
fragment, it SHALL be a four-item array, in this order:

1. Source node ID.
2. Creation timestamp.
3. Fragment offset.
4. Fragment payload length.

Fragment offset and payload length MUST appear together, and MUST
appear if and only if the subject bundle is a fragment. Offset zero does
not mean "not a fragment". The length is the length in bytes of the
received fragment's payload, not the total ADU length.

Source node IDs SHALL use the applicable BP EID encoding of {{!RFC9171,
Section 4.2.5}} and the referenced URI scheme specification. The null
endpoint ID is not permitted. Creation timestamps SHALL use {{!RFC9171,
Section 4.2.7}}. Fragment offsets and payload lengths SHALL be CBOR
unsigned integers. Reception times SHALL be DTN times as defined in
{{!RFC9171, Section 4.2.6}}, including its zero value for unknown time.

An entry MUST include a reception time if the subject bundle's "Report
status time" bundle processing control flag is set. Otherwise, it MUST
omit the reception time. This is the reception event's time as reported
by the receiving node's local clock, not the time of batch creation or
transmission. A
batch MAY mix entries with and without reception times, and MAY contain
different source node IDs and creation timestamps. All entries MUST
have the same resolved requesting administrative endpoint ID. Resolve this
ID as specified in {{previous-node}} and compare it as specified in
{{eid-comparison}}. That endpoint is the destination of the carrier bundle.
The reporting and requesting node IDs are carried by that bundle, not
repeated in each entry.

## CDDL {#cddl}

The following CDDL {{!RFC8610}} defines the record structure. TBD2 is a
symbolic placeholder for the single record type code to be allocated by
IANA. The temporary `uint` definition lets tools use the CDDL before
allocation. It does not permit arbitrary record type codes on the wire.
The EID rule describes the common BP EID structure. Its scheme-specific
part MUST conform to the applicable URI scheme encoding. The EID MUST
identify a non-anonymous source node. The relationship
to the subject bundle's fragment and status-time flags is specified in
{{record}} and cannot be checked from the record structure alone.

~~~ cddl
hop-reception-report = [TBD2, reception-batch]
TBD2 = uint ; Replace with the allocated unsigned integer literal.

reception-batch = [1* reception-entry]
reception-entry = [bundle-id, ? reception-time]

bundle-id = [source-node-id, creation-timestamp]
          / [source-node-id, creation-timestamp,
             fragment-offset, fragment-payload-length]

source-node-id = [uri-scheme-code: uint, scheme-specific-part: any]
creation-timestamp = [creation-time: dtn-time, sequence-number: uint]
dtn-time = uint
reception-time = dtn-time
fragment-offset = uint
fragment-payload-length = uint
~~~

# Node Processing

## Requesting Node {#forwarding}

A node MAY insert a Hop Reception Report Request block when forwarding a
bundle to a neighboring node. The bundle source MAY also insert a request
on initial transmission, subject to the exclusions below. Local policy
controls this decision. It MAY consider the bundle's characteristics,
the next hop's identity, and the convergence layer in use.

If the requesting node includes a Previous Node block, it MUST place its
own administrative endpoint ID in that block, as specified in
{{previous-node}}. Absence of a Previous Node block does not prohibit
inserting a request. The bundle source MUST NOT insert a Previous Node
block contrary to {{!RFC9171, Section 4.4.1}}. On initial transmission, the
first receiving BPA can honor a request without that block through the
authenticated or configured peer-association fallback in {{previous-node}}.
This does not change RFC 9171's Previous Node processing requirements.

A sender cannot assume that the receiver has sufficient identity
information or will honor the request. Absence of a report remains
inconclusive.

A node MUST NOT insert a Hop Reception Report Request block into a bundle
whose payload is an administrative record, or whose source node ID is the
null endpoint ID. These exclusions match the status report request
restrictions in {{!RFC9171, Section 4.2.3}}. They prevent reports from
triggering further reception reports.

If a received bundle still contains a Hop Reception Report Request block,
the node MUST remove that block before inserting its own. Removal or
replacement of protected hop-specific blocks MUST follow the security
processing in {{request-security}}.

## Receiving Node {#receiving}

A node that implements this specification SHALL associate each received
request with the reception event at Step 2 of {{!RFC9171, Section 5.6}}.
This is the event that a source-requested reception status report would
report. The node MUST check the following before honoring the request:

1. The bundle contains exactly one Hop Reception Report Request block,
   with the structure specified in {{block}} and the required block
   processing control flag settings in {{flags}}. Unrecognized bits in
   the block-type-specific flags field are ignored as specified in
   {{block}}.
2. The subject bundle is neither an administrative-record bundle nor an
   anonymous-source bundle. A node MUST NOT generate a Hop Reception
   Report in response to a Hop Reception Report, even if a request block
   is present.
3. The requesting identity and administrative endpoint are established
   under {{previous-node}} through one of two cases:

   - Exactly one valid Previous Node block identifies the requesting
     administrative endpoint and is consistent with the established
     sending-peer identity.
   - No Previous Node block is present, and convergence-layer peer
     authentication or trusted configuration establishes both the
     immediate sending BPA's identity and its administrative endpoint for
     this reception.

   A malformed, duplicated, or inconsistent Previous Node block does not
   qualify for the second case. EID comparisons and associations with the
   actual sending peer MUST follow {{eid-comparison}} and {{previous-node}}.
4. The node can determine the subject bundle's identifier and status-time
   flag as specified in {{record}}.

These checks establish request validity and eligibility. They do not
assert that all later reception validation has succeeded.

The node MUST capture the resolved requesting administrative endpoint and
its association with the immediate sender before any of these events:

- Reception context is discarded.
- The CLA session changes.
- Hop-specific blocks are removed or replaced.
- The bundle is forwarded or reassembled.

Later session state MUST NOT redirect a pending report. Before reassembly
or changes to hop-specific blocks, the node MUST also capture:

- The subject bundle or fragment identity.
- The status-time flag.
- The reception event's time, when required.

The node MUST retain the local elapsed-time basis for the aggregation
deadline, measured from reception ({{batching}}). Delayed request checks
or batch construction MUST NOT change the reported event or replace its
timestamp with the time of those later actions.

Local policy MAY permit the node to honor an otherwise valid request. It
cannot make an invalid request valid. For each honored request, the node
SHALL create a pending reception entry as specified in {{record}}.
Batching and reporting follow {{batching}}. It MUST NOT delay
processing or forwarding of the subject bundle while waiting to fill a
batch.

If a request is invalid, the node MUST NOT create a reception entry for
it. The node SHALL treat the request block as one that cannot be
processed, subject to the malformed-block and unprocessable-block rules
of {{!RFC9171, Section 5.6}}. The node cannot honor a request
without the required sender association or with a conflicting Previous
Node block. Neither condition introduces a new subject-bundle deletion
rule.
A policy decision to decline an otherwise valid request does not itself
invalidate or delete the subject bundle.

A node that supports this specification MUST remove all received request
blocks before onward forwarding. This applies to honored, declined, and
invalid requests, regardless of the "Discard block if it can't be
processed" flags. Applicable security processing MUST precede removal or
replacement of protected blocks, as specified in {{request-security}}.
The node MAY insert its own new request as described in {{forwarding}}.
Ordinary source-requested status reporting remains independent
({{status-reports}}).

## Batching and Scheduling {#batching}

A reporting node MUST batch only pending entries that meet both conditions:

- They have the same resolved requesting administrative endpoint ID
  ({{previous-node}}), compared as specified in {{eid-comparison}}.
- They have compatible security treatment.

Entries resolved using a Previous Node block and entries resolved using
the fallback MAY share a batch when both conditions hold. The carrier's
authentication, integrity, confidentiality, and security-policy context
must satisfy each entry's requirements. Batching MUST NOT weaken required
protection for any entry. Different
subject source EIDs and creation timestamps do not prevent batching.
Single-entry reports are permitted.

Implementations MUST provide configurable, finite limits for:

- Pending reporting state.
- Batch size, by entry count and encoded size.
- Maximum aggregation delay.

Size limits MUST account for the carrier bundle and
its security overhead within applicable bundle-size limits. These are
local resource limits, not fixed protocol maxima.

The node MUST measure each entry's maximum aggregation delay from the
reception event in {{receiving}}, using local elapsed time. An accurate
DTN clock is not required. Later creation of pending state or addition of
entries MUST NOT extend that deadline.

A flush ends aggregation for selected entries and puts them in one or
more report carrier bundles. It makes those reports available to the
local transmission scheduler.

Unless the resource policy below discards the pending state, a node MUST
flush entries when an applicable size threshold, count threshold, or
maximum aggregation delay is reached. If the node honors an eligible
request at or after its aggregation deadline, it MUST flush the entry
without further aggregation delay. The same resource policy applies.
Reaching the deadline does not guarantee immediate transmission. An
unavailable contact can still delay transmission.

A node MAY flush earlier under local scheduling policy. For example, it
can flush a single entry immediately or use a return contact before it
ends.
Batching over a continuously available return link is permitted. Link
availability alone does not require immediate flushing or transmission of
every new entry. Subject-bundle processing and forwarding MUST NOT wait
for a batch to fill.

Under resource pressure, local policy MAY decline new requests or discard
pending reporting state, including entries previously created for honored
requests. Neither action changes processing of the subject bundle or
creates a negative reception assertion. Nodes need not retain reporting
state or queued report bundles without limit.

Batching reduces the bundle overhead per entry. However, it increases
acknowledgement delay and the number of assertions lost with one report
bundle. Report lifetime, scheduling, routing, and retry policy are local
matters. Report delivery is not guaranteed.

## Carrier Bundle Construction {#carrier}

For each outgoing batch, the administrative element SHALL construct one
Hop Reception Report administrative record and request its transmission
in a carrier bundle ({{!RFC9171, Section 6.2}}). The carrier's payload SHALL
contain exactly that one record, including its two-element envelope and
nonempty batch. The carrier's source node ID MUST identify the reporting
node. Its destination MUST be the resolved requesting administrative
endpoint ID for all entries. This is the ID determined by
{{previous-node}} and captured under {{receiving}}. Endpoint comparison
follows {{eid-comparison}}. Those identities MUST NOT be added to each
entry.

The carrier MUST have the "ADU is an administrative record" bundle
processing control flag set. It MUST comply with the administrative-bundle
restrictions of {{!RFC9171}}. In particular, all status report request
flags MUST be zero (Section 4.2.3), and the "Transmit status report if
block can't be processed" flag MUST be zero in every canonical block
(Section 4.2.4). The carrier MUST NOT contain a Hop Reception Report
Request block. Its destination identifies the requesting neighbor but
does not restrict the return route to a single link.

## Report Recipient Processing {#report-processing}

The receiving administrative element MUST validate the complete Hop
Reception Report record before applying any entry. Validation includes:

- The two-element envelope and supported record type.
- A nonempty batch.
- Each entry's structure and field encodings.
- The paired fragment fields.

An empty batch, incomplete fragment identity, or malformed entry
invalidates the entire record. A valid prefix MUST NOT be applied.

The administrative element SHALL ignore malformed administrative record
ADUs and record types it cannot process, as required by {{!RFC9713,
Section 2}}. It MUST also ignore records that violate this specification's
format. This does not undo delivery of the carrier bundle or change
its BPA processing. Implementations MAY limit the resources used for
validation. They MUST NOT partially apply a record they could not validate
completely.

Ignoring an unsupported record type is compatibility behavior, not
permission to report to an uncertain destination. A sender of a report
MUST still establish its recipient under {{previous-node}}.

The recipient MUST correlate each entry with locally tracked
transmissions of that bundle or fragment to the reporting neighbor. The
carrier's source node ID identifies that neighbor. Subject-source EID
matching and reporting-node and peer-identity correlation MUST follow
{{eid-comparison}}. Before using a report to decide whether to retain or
retransmit a bundle, the recipient MUST establish the reporting node's
identity and the integrity of the reported entries ({{security}}).
A bundle source field alone is not authentication.

The recipient MUST process duplicate assertions idempotently, within or
across reports. Repeated assertions must not produce additional effects.
Unknown identifiers MUST NOT acknowledge other bundles or fragments. The
recipient MAY ignore those entries without invalidating useful entries
in the validated record.

When local subject information is available, an entry MUST NOT
acknowledge a subject if its fragment identity or timestamp presence is
inconsistent with that information. Identifiers name bundles or fragments,
not individual retransmission attempts. A delayed or replayed assertion
cannot establish which attempt was received or whether the reporting node
still retains the bundle.

## Interaction with Fragmentation {#fragmentation}

If a node fragments a bundle with a request block, bit 0 controls
replication of that block. When bit 0 is set, as recommended in {{flags}},
the node replicates the block in each fragment. Each honored
fragment request creates an entry carrying that received fragment's
identity, including its offset and payload length. Multiple such entries
MAY be batched together if they meet {{batching}}, even if their fragments
are subsequently reassembled.

The node MUST capture the requester and fragment identity before
reassembly or replacement or removal of hop-specific blocks, as required
in {{receiving}}. Reassembly MUST NOT substitute a non-fragment identifier
for a bundle that was not received in that form. A report that acknowledges
one fragment does not acknowledge the entire original bundle. Fragments
from different requesting nodes cannot share a batch addressed to just
one of them.

# Relationship to Other Mechanisms

## Bundle Status Reports {#status-reports}

This document does not change the Bundle Status Report format or the
rules for reports requested by primary-block flags. Ordinary
source-requested reporting remains unchanged and independent of Hop
Reception Reports. The two mechanisms
use different administrative record types, even when their destinations
are the same. An ordinary Bundle Status Report does not satisfy a hop request,
and a Hop Reception Report does not satisfy a source-requested status
report.

The traffic concerns of {{!RFC9171, Section 5.1}} also apply here. Hop
Reception Report generation MUST be disabled by default. It MUST be
enabled only when the risk of excessive traffic is deemed acceptable.
Each honored request creates a pending entry, not necessarily a separate
report bundle.
Batching shares carrier overhead across entries. Requests still use
processing, storage, and return-path capacity.

The requesting node controls when it requests reports. The reporting node
controls whether to honor requests and how to batch and schedule reports.
Return traffic need not use the immediate link.

## Custody Transfer

This document does not define custody transfer. A report tells the
requesting node only that the next hop's BPA received a bundle or fragment.
The receiving node makes no commitment to forward, deliver, or retain it.
The request adds no responsibility beyond {{!RFC9171}}. Local policy
determines how the requesting node uses the report.

## Convergence Layer Completion Indications

Convergence-layer specifications usually define a "transmission success"
indication to the BPA. Hop Reception Reports supplement these indications;
they do not replace them. They explicitly assert BPA reception,
independent of the convergence layer in use.

Even after convergence-layer success, a missing report does not show
whether the receiver lacks support or the transfer failed. Other causes
of silence include:

- Insufficient sender-identity information.
- Policy refusal.
- Discarded reporting state.
- Pending aggregation.
- An unavailable return contact.
- Receiver failure.
- Report loss.

Neither silence nor a missing entry is a negative assertion about
reception.

# Examples {#examples}

These examples show the administrative-record wrapper in CBOR diagnostic
notation. The token `TBD2` represents the unsigned integer record type
code that IANA will allocate. It is not a CBOR text string.

The record-format examples use these carrier addresses:

- Source: reporting node `dtn://receiver/`.
- Destination: requesting administrative endpoint `dtn://forwarder/`,
  resolved under {{previous-node}}.

These two identities do not appear in the entries. All subject requests in each
batch came from that requesting node and have compatible security
treatment.

## One Unfragmented Bundle

The subject source is `dtn://source-a/`. The creation timestamp is
`[1000, 0]`. The subject bundle did not request status time:

~~~ cbor-diag
[TBD2, [
  [[[1, "//source-a/"], [1000, 0]]]
]]
~~~

## Different Sources and Mixed Timestamp Presence

These unfragmented bundles have different sources and creation
timestamps. Only the first subject requested status time. Only its entry
contains a reception time: 3000 milliseconds after the DTN epoch.

~~~ cbor-diag
[TBD2, [
  [[[1, "//source-a/"], [1000, 0]], 3000],
  [[[1, "//source-b/"], [2000, 1]]]
]]
~~~

## Multiple Fragments, Including Offset Zero

These are two 512-byte fragments of a 1024-byte ADU from
`dtn://source-a/`, with the same creation timestamp `[4000, 0]`. Both
requested status time. The first fragment starts at offset zero, but
still includes both fragment fields. Neither entry carries the total ADU
length, and each asserts only reception of its own fragment:

~~~ cbor-diag
[TBD2, [
  [[[1, "//source-a/"], [4000, 0], 0, 512], 5000],
  [[[1, "//source-a/"], [4000, 0], 512, 512], 5100]
]]
~~~

## Invalid Records

An empty batch is invalid:

~~~ cbor-diag
[TBD2, []]
~~~

An identifier containing an offset but no fragment payload length is also
invalid, including when the offset is zero:

~~~ cbor-diag
[TBD2, [
  [[[1, "//source-a/"], [4000, 0], 0], 5000]
]]
~~~

Both invalid records MUST be ignored as described in {{report-processing}}.

## Initial Transmission without a Previous Node Block

The source BPA `dtn://source-a/` initially transmits an eligible,
unfragmented bundle with:

- Creation timestamp `[1000, 0]`.
- A Hop Reception Report Request block.
- No Previous Node block.
- No request for status time.

The receiving BPA `dtn://receiver/` uses an authenticated CLA association
or trusted configuration that applies to this transfer. It establishes
that the immediate sender is the source BPA and that its administrative
endpoint is `dtn://source-a/`. The primary block's source node ID alone
does not establish this association.

If local policy permits the request, the receiver captures the association
and creates the entry `[[[1, "//source-a/"], [1000, 0]]]`. It can send a
single-entry report with source `dtn://receiver/` and destination
`dtn://source-a/`, using the wrapper shown above. It can also batch the
entry with others for that endpoint if their security treatment is
compatible. Without the required association, it does not honor the
request.

## Conflicting Previous Node Identity

The established immediate sender is `dtn://forwarder/`. A Previous Node
block names `dtn://other/`. Authentication or trusted configuration does
not associate that identity with the sender.

The receiver does not honor the request. It cannot ignore the conflicting
block and use fallback to report to either endpoint. RFC 9171 handling of
malformed or unprocessable blocks still applies. This conflict introduces
no new subject-bundle deletion rule.

# Security Considerations {#security}

## Reflection

Requests can be used for reflection attacks. An attacker could send a
bundle with a request block and a Previous Node block that names a victim.
Without sender validation, the receiver could send a report to the victim.

The main protection is the sender association required by
{{previous-node}} and {{receiving}}. It connects the requesting
administrative endpoint to the BPA that supplied the subject bundle. When
a Previous Node block is present, that association MUST be consistent
with the block.
When the block is absent, authenticated CLA information or trusted
configuration can establish the association. Fallback cannot bypass a
malformed, duplicated, or conflicting block.

Transport-peer authentication alone does not establish which BPA and
administrative endpoint the peer represents. TCPCLv4 peer-node
authentication {{?RFC9174}} can help establish this association when it
succeeds under an appropriate trust policy. A TCPCL session alone does
not authenticate an advertised node ID. Trusted configuration can instead
bind a controlled link or peer to a BPA and administrative endpoint.
In either case, the association must apply to the subject bundle's
reception, not merely to a later session.
A transport address, unauthenticated advertised identity, or primary-block
source node ID alone does not suffice.

A node MUST NOT honor the request if the required association cannot be
established. Administrative endpoint syntax alone proves neither peer
identity nor authorization. EID comparisons and any association between
different node identities follow {{eid-comparison}}. Request
authentication does not replace the actual-sender association check.
Knowledge of a peer's identity does not require the receiver to honor its
requests. Local policy MAY decline even a valid request.

Addressing a report to the previous hop does not limit its route or
reflection costs to the immediate link. Local policy controls return
routing. Authorization, rate limits, and resource limits remain necessary,
even for authenticated requesting peers.

## Integrity and Reporter Authentication {#request-security}

The requesting node can use a Block Integrity Block (BIB) {{!RFC9172}} to
protect its request block and any accompanying Previous Node block. The
requesting node is the security source. With the appropriate security
association and policy, the receiver can verify who inserted the request.

Source-applied protection cannot cover blocks inserted later by a transit
node. A source can protect a request that it inserts on initial
transmission. The security-acceptor processing below applies in both cases.

For BIB operations on the request block or accompanying Previous Node
block, security policy MUST designate the intended next hop as security
acceptor. The receiving node
MUST perform the applicable security processing before removing or
replacing the protected blocks. Following {{!RFC9172, Section 5.1.2}}, the
acceptor MUST remove operation-specific information from the BIB. This
includes the target and corresponding security results. The acceptor MUST
remove the BIB when no security operations remain. It MUST NOT remove
unrelated security operations merely to remove hop-specific blocks.
Those operations remain subject to their own security processing.
Removal or replacement MUST NOT leave references to missing security
targets.

The node MUST handle verification failures under the applicable security
policy. It MUST NOT treat failed protection as successful request
authentication. These requirements apply existing BPSec procedures. They
define neither a new security context nor a stronger reception assertion.

Authentication of the request or incoming subject-bundle transfer does
not automatically authenticate a later report, including one returned
over another path. A forged or modified report could cause early deletion
of a retained copy or prevent a needed retransmission.

Before using a report to decide whether to retain or retransmit a bundle,
the recipient MUST establish the reporter's identity and the integrity of
all entries it uses. This includes their binding to that identity. BPSec
can provide protection through an appropriate security context and trust
policy. That protection covers the report payload and binds it to the
carrier's source identity. All entries must have compatible security
requirements before batching ({{batching}}).

A direct convergence-layer exchange can provide this assurance if it
authenticates the reporting peer and protects the report's integrity.
On a multihop return path, authentication of the last transport peer does
not authenticate the original reporter. It also does not protect entries
against changes by earlier nodes. Such paths need protection and trust
arrangements that establish the original reporter's identity and the
integrity of its assertions. The carrier source field alone is not enough.
When bundle-layer security services are needed, they SHALL use BPSec as
required by {{!RFC9171, Section 8}}. Authentication establishes who made an
assertion. It does not prove that a dishonest reporter received or
retained the subject.

## Replay and Duplicates

A valid report can be duplicated or replayed. Recipients MUST apply
assertions idempotently and correlate them with locally tracked bundle or
fragment identities and the reporting neighbor ({{report-processing}}).
Neither a bundle identifier nor an optional reception time identifies an
individual retransmission attempt. A replayed report is not evidence of a
new reception or continued retention. To decide whether an
assertion is useful, local policy needs to consider stale reports, limits
on correlation state, and clock limitations. This specification adds no
sequence-number or anti-replay exchange.

## Resource Exhaustion and Correlated Loss

Requests can consume pending-state storage, encoding work, security
processing, and return-path capacity, even when peers are authenticated.
Large received batches also consume validation and correlation resources.
Nodes SHOULD apply per-peer rate limits and resource budgets in addition
to the configurable limits in {{batching}}. It is better to decline
requests or discard pending state than to exhaust resources needed for
subject-bundle processing. A request does not guarantee a
report, and a refusal is not encoded as a negative entry.

Loss or suppression of one batched report removes multiple assertions at
once. This correlated loss can cause bursts of unnecessary retransmission
if local policy reacts to silence. Batch-size limits, delay limits,
scheduling, and retransmission policies should account for this effect.
A missing entry does not establish an independent transfer failure.

## Traffic Analysis

Reports expose bundle identifiers and, when requested, reception times.
A batch can link traffic from different sources through a common
requesting and reporting node. Its size, membership, and timing can reveal
traffic volumes and contact patterns. Aggregation also means
that report transmission timing is not simply the reception event time.

Convergence-layer confidentiality, such as TCPCLv4 with TLS, protects
content on links where it is used. It does not hide all traffic metadata
or protect content from intermediate nodes on a multihop return path.
BPSec payload confidentiality can protect entries across that path.
Carrier addresses, sizes, and timing can remain visible.
Confidentiality policies for different subjects must not be weakened by
placing them in a common batch.

# IANA Considerations {#iana}

IANA is requested to assign a value from the "Bundle Block Types"
subregistry of the "Bundle Protocol" registry, in the range for Bundle
Protocol Version 7, as follows:

| Bundle Protocol Version | Value | Description | Reference |
|-------------------------|-------|-------------|-----------|
| 7 | TBD1 | Hop Reception Report Request | This document |
{: #tab-block-type align="left" title="Bundle Block Type Registration"}

IANA is also requested to assign a value from the "Bundle Administrative
Record Types" registry within the "Bundle Protocol" registry group, as
specified by {{!RFC9713}}, as follows:

| Bundle Protocol Version | Value | Description | Reference |
|-------------------------|-------|-------------|-----------|
| 7 | TBD2 | Hop Reception Report | This document |
{: #tab-record-type align="left" title="Administrative Record Type Registration"}

TBD1 and TBD2 are placeholders in distinct registries, not allocated
numbers. TBD2 does not reuse ordinary Bundle Status Report type 1. The
extension-block range 192-255 for Private and/or Experimental Use is not
an administrative-record experimental range. For BPv7 administrative
records, {{!RFC9713}} reserves 64384-64511 for Experimental Use and
64512-65535 for Private Use. This document does not select an experimental
code point for either placeholder.

--- back

# Acknowledgments
{:numbered="false"}

Discussion of QUIC Bundle Protocol Convergence Layer completion semantics
motivated this work. Convergence-layer completion does not necessarily
establish BPA reception. Its guarantees depend on the convergence-layer
specification and implementation. Hop Reception Reports
provide explicit, convergence-layer-independent BPA reception assertions.
