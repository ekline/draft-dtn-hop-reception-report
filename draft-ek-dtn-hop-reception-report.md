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

This document defines the Hop Reception Report Request block, a Bundle
Protocol version 7 (BPv7) extension block by which a forwarding node
requests that the next-hop node report reception of the bundle back to it.
The request is addressed implicitly, using the node ID already carried in
the bundle's Previous Node block. This document also defines the Hop
Reception Report administrative record, which batches positive reception
assertions for one or more bundles or fragments for the same requesting
neighbor. These reports do not transfer custody or responsibility for the
reported bundles.

--- middle

# Introduction {#intro}

When a Bundle Protocol version 7 (BPv7) {{!RFC9171}} node forwards a bundle
to a neighboring node, the Convergence Layer Adapter (CLA) it uses can
typically indicate that the transfer completed. That indication, however,
reflects only what the convergence layer or underlying transport can
observe: that the data reached the peer's convergence layer (or, in some
cases, only the peer's transport stack). It does not indicate that the
peer's Bundle Protocol Agent (BPA) has received the bundle. If the peer's convergence layer or BPA fails between
transport-level receipt and BPA processing, the bundle may be lost while
the forwarding node believes it was successfully transferred.

BPv7 provides Bundle Status Reports ({{!RFC9171, Section 6.1.1}}) as a
mechanism for nodes to report the reception, forwarding, delivery, or
deletion of a bundle. However, whether reports are requested, and where
they are sent, is determined solely by the bundle's source via the primary
block, which is immutable after creation. A source can request reception
reports from nodes along the path, subject to their reporting policies,
but a forwarding node cannot use those primary-block flags to request a
report for its own purposes.

This document defines the Hop Reception Report Request block, an extension
block that a forwarding node inserts to request that the next-hop node
send a reception report back to the forwarding node. The block carries no
addressing information of its own: the report is sent to the validated
node ID in the bundle's Previous Node block ({{!RFC9171, Section 4.4.1}}).
The response uses a new administrative record type, Hop Reception Report,
containing a nonempty batch of positive reception assertions. Ordinary
Bundle Status Reports are already administrative records; the new type
provides different content and batching semantics, rather than changing
the administrative-record flag on an ordinary status report. The request
remains an extension block in the subject bundle, not a separate
administrative-record exchange.

The resulting signal, "the next-hop BPA has received this bundle", has
exactly the meaning of the "reporting node received bundle" status
assertion of {{!RFC9171, Section 6.1.1}}. That assertion is generated at
Step 2 of {{!RFC9171, Section 5.6}}, before subsequent validation. It does
not establish successful later validation, completion of all reception
processing, durable storage, custody, forwarding, delivery, or continued
retention. It is an explicit BPA-level indication independent of the
convergence layer, not a claim that every convergence-layer completion
indication is weaker. How a forwarding BPA uses this signal is a matter
of local policy and is outside the scope of this document. In particular,
this document does not define a custody transfer mechanism, and the report
does not transfer any responsibility for the bundle between nodes.

"One hop" describes the reception event and the requesting neighbor.
Supporting receiving nodes remove the request block before onward
forwarding, and the required block processing control flags cause
non-supporting nodes to discard it. Addressing the report carrier bundle
to the requesting neighbor does not constrain its route to the immediate
link. Return routing is a matter of local policy.

## Open Issues {#open-issues}

RFC EDITOR: Please remove this section before publication.

The following point is noted for working group discussion:

Reception timestamp:
: This revision includes a reception time in an entry if and only if the
  subject bundle's "Report status time" flag is set. A future revision
  could allow the forwarding node to request timestamps independently of
  that source-controlled flag. No such request flag is defined here.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

BPA:
: Bundle Protocol Agent, as defined in {{!RFC9171}}.

CLA:
: Convergence Layer Adapter, as defined in {{!RFC9171}}.

Forwarding node:
: The node that inserts a Hop Reception Report Request block into a bundle
  prior to forwarding it.

Receiving node:
: The node to which the forwarding node forwards the bundle, and which
  processes the Hop Reception Report Request block.

# Hop Reception Report Request Block {#block}

## Block Definition

The Hop Reception Report Request block is a BPv7 canonical block with a
block type code of TBD1 (see {{iana}}).

The block-type-specific data of the block SHALL be a single CBOR unsigned
integer {{!RFC8949}} representing a flags field. No flags are defined by
this document; the value SHALL be zero (0) when emitted, and receiving nodes
SHALL ignore any flag bits they do not recognize.

The block SHALL NOT appear more than once in a bundle.

## Dependence on the Previous Node Block {#previous-node}

The Hop Reception Report Request block has meaning only in conjunction with
a Previous Node block ({{!RFC9171, Section 4.4.1}}), which identifies the
node to which the reception report is to be sent. A bundle containing a Hop
Reception Report Request block but no Previous Node block MUST be treated
by the receiving node as if the Hop Reception Report Request block could
not be processed (see {{receiving}}).

Because the Previous Node block is itself replaced at each hop, the
addressing of the request follows the bundle's actual path without the
forwarding node needing to place its own node ID in the request block.

## Block Processing Control Flags {#flags}

A forwarding node inserting the Hop Reception Report Request block SHALL
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
applicable bundle-size and implementation limits; this specification
imposes no fixed protocol maximum and does not require unbounded
resources. Each entry positively asserts reception of the identified
bundle or fragment. Omission of an identifier conveys no negative
assertion. The record has no status indicators, ordinary Bundle Status
Report status arrays, or reason codes, and conveys no custody, refusal,
forwarding, delivery, or deletion status.

Each entry SHALL be an array containing a bundle identifier and, when
required below, a reception time. The identifier SHALL be an array of two
items for a non-fragment: its source node ID and creation timestamp. For a
fragment it SHALL instead contain four items: the source node ID,
creation timestamp, fragment offset, and fragment payload length, in that
order. Fragment offset and payload length MUST appear together, and MUST
appear if and only if the subject bundle is a fragment. Offset zero does
not mean "not a fragment". The length is the length in bytes of the
received fragment's payload, not the total ADU length.

Source node IDs SHALL use the applicable BP EID encoding of {{!RFC9171,
Section 4.2.5}} and the referenced URI scheme specification; the null
endpoint ID is not permitted. Creation timestamps SHALL use {{!RFC9171,
Section 4.2.7}}. Fragment offsets and payload lengths SHALL be CBOR
unsigned integers. Reception times SHALL be DTN times as defined in
{{!RFC9171, Section 4.2.6}}, including its zero value for unknown time.

An entry MUST include a reception time if the subject bundle's "Report
status time" bundle processing control flag is set, and MUST omit it
otherwise. This is the reception event's time as reported by the receiving
node's local clock, not the time of batch creation or transmission. A
batch MAY mix entries with and without reception times, and MAY contain
different source node IDs and creation timestamps. All entries MUST
concern requests from the same validated previous-hop node, which is the
recipient of the carrier bundle. The reporting and requesting node IDs
are carried by that bundle, not repeated in each entry.

## CDDL {#cddl}

The following CDDL {{!RFC8610}} defines the record structure. TBD2 is a
symbolic placeholder for the single record type code to be allocated by
IANA. Its temporary definition as `uint` makes the CDDL usable before
allocation; it does not permit arbitrary record type codes on the wire.
The EID rule describes the common BP EID structure; its scheme-specific
part MUST additionally conform to the applicable URI scheme encoding,
and the EID MUST identify a non-anonymous source node. The relationship
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

## Forwarding Node {#forwarding}

A node MAY insert a Hop Reception Report Request block into a bundle when
it forwards the bundle to a neighboring node. The decision to do so is a
matter of local policy, which MAY take into account the bundle's
characteristics, the identity of the next hop, and the convergence layer in
use.

When inserting the block, the forwarding node MUST also ensure the bundle
contains a Previous Node block whose node ID identifies the forwarding node,
as required by {{!RFC9171, Section 4.4.1}}. A node MUST NOT insert a Hop
Reception Report Request block into a bundle that does not, or will not,
contain a Previous Node block.

A node MUST NOT insert a Hop Reception Report Request block into a bundle
whose payload is an administrative record, or whose source node ID is the
null endpoint ID. This parallels the restrictions on status report request
flags in {{!RFC9171, Section 4.2.3}} and prevents reception reports from
themselves generating reception reports.

If a bundle received by the node already contains a Hop Reception Report
Request block that was not removed (for example, because the node received
the bundle from a node that does not implement this specification and did
not honor the "discard block" flag), the node MUST remove it before
inserting its own.

## Receiving Node {#receiving}

Upon receiving a bundle containing a Hop Reception Report Request block,
a node that implements this specification SHALL associate the request with
the reception event at Step 2 of {{!RFC9171, Section 5.6}}, at which a
source-requested reception status report would be generated. The node
MUST check the following before honoring the request:

1. The bundle contains exactly one Hop Reception Report Request block,
   with the structure specified in {{block}} and the required block
   processing control flag settings in {{flags}}. Unrecognized bits in
   the block-type-specific flags field are ignored as specified in
   {{block}}.
2. The subject bundle is neither an administrative-record bundle nor an
   anonymous-source bundle. A node MUST NOT generate a Hop Reception
   Report in response to a Hop Reception Report, even if a request block
   is present.
3. The bundle contains exactly one well-formed Previous Node block with a
   valid node ID.
4. The Previous Node ID identifies the actual convergence-layer peer from
   which this bundle was received. The node MUST establish this
   correspondence through peer authentication or trusted configuration;
   absent that association, it MUST NOT honor the request ({{security}}).
5. The subject bundle's identifier and status-time flag can be determined
   in accordance with {{record}}.

These are request validity and eligibility checks, not an assertion that
all subsequent reception validation has succeeded. The node MUST capture
the requesting node ID, the subject bundle or fragment identity, the
status-time flag, and, when required, the reception event's time before
reassembly or replacement or removal of hop-specific blocks. Deferring
request checks or batch construction MUST NOT change the event being
reported or replace its timestamp with the time of those later actions.

Local policy MAY permit honoring an otherwise valid request; it cannot
make an invalid request valid. For each request it honors, the node SHALL
create a pending reception entry as specified in {{record}}, for
aggregation and reporting according to {{batching}}. It MUST NOT delay
processing or forwarding of the subject bundle while waiting to fill a
batch.

If a request is invalid, the node MUST NOT create a reception entry for
it. The node SHALL treat the request block as one that cannot be
processed, subject to the malformed-block and unprocessable-block rules
of {{!RFC9171, Section 5.6}}. A policy decision to decline an otherwise
valid request does not itself invalidate or delete the subject bundle.

Whether the request is honored, declined, or invalid, a supporting node
MUST remove all received Hop Reception Report Request blocks before
onward forwarding, regardless of their "Discard block if it can't be
processed" flags. The node MAY insert its own new request as described
in {{forwarding}}. Ordinary source-requested status reporting remains
independent ({{status-reports}}).

## Batching and Scheduling {#batching}

A reporting node MUST batch only pending entries with the same validated
recipient node ID and compatible security treatment. Compatibility means
that the carrier's authentication, integrity, confidentiality, and
security-policy context satisfy the requirements for every entry;
batching MUST NOT weaken protection required for any entry. Different
subject source EIDs and creation timestamps do not prevent batching.
Single-entry reports are permitted.

Implementations MUST provide configurable finite bounds on pending
reporting state, batch size (entry count and encoded size), and maximum
aggregation delay. Size limits MUST account for the carrier bundle and
its security overhead within applicable bundle-size limits. These are
local resource limits, not fixed protocol maxima. The aggregation delay
for an entry begins when the pending entry is created; adding entries
MUST NOT postpone an older entry's deadline.

Unless pending state is discarded under the resource policy below, a node
MUST flush pending entries into one or more report carrier bundles when
an applicable size or count threshold is reached, when the maximum
aggregation delay is reached, or when an appropriate return-contact
opportunity occurs. A flush ends aggregation for those entries and makes
the resulting reports available to the local transmission scheduler.
Expiration of the aggregation delay does not guarantee immediate
transmission over an unavailable contact. A node MAY flush earlier,
including immediately for a single entry.

Under resource pressure, local policy MAY decline new requests or discard
pending reporting state, including entries previously created for honored
requests. Neither action changes processing of the subject bundle or
creates a negative reception assertion. Reporting state and queued report
bundles need not be retained without bound.

Batching reduces per-report bundle overhead, but increases acknowledgement
latency and the number of reception assertions lost with one report
bundle. Report lifetime, scheduling, routing, and retry policy are local
matters. Report delivery is not guaranteed.

## Carrier Bundle Construction {#carrier}

For each batch to be sent, the administrative element SHALL construct one
Hop Reception Report administrative record and request transmission of a
carrier bundle as in {{!RFC9171, Section 6.2}}. The carrier's payload SHALL
contain exactly that one record, including its two-element envelope and
nonempty batch. The carrier bundle's source node ID MUST identify the
reporting node, and its destination MUST be the validated requesting node
ID shared by all entries. Those identities MUST NOT be added to each
entry.

The carrier MUST have the "ADU is an administrative record" bundle
processing control flag set and MUST comply with the administrative-bundle
restrictions of {{!RFC9171}}. In particular, all status report request
flags MUST be zero (Section 4.2.3), and the "Transmit status report if
block can't be processed" flag MUST be zero in every canonical block
(Section 4.2.4). The carrier MUST NOT contain a Hop Reception Report
Request block. Its destination identifies the requesting neighbor but
does not restrict the return route to a single link.

## Report Recipient Processing {#report-processing}

The receiving administrative element MUST validate the complete Hop
Reception Report record before applying any entry. This includes the
two-element envelope and supported record type, a nonempty batch, every
entry's structure and field encodings, and the paired fragment fields.
An empty batch, incomplete fragment identity, or malformed entry
invalidates the entire record; a valid prefix MUST NOT be applied.

As required by {{!RFC9713, Section 2}}, a not-well-formed administrative
record ADU or a record type that the element cannot process SHALL be
ignored. Records that violate this specification's format MUST likewise
be ignored. This does not undo delivery of the carrier bundle or change
its BPA processing. Implementations MAY impose resource limits when
validating records, but MUST NOT partially apply a record they could not
validate completely.

The recipient MUST correlate each entry with locally tracked
transmissions of the identified bundle or fragment to the reporting
neighbor identified by the carrier's source node ID. Before using a
report for consequential retention or retransmission decisions, the
recipient MUST establish the reporting node's identity and the integrity
of the reported entries as described in {{security}}. A bundle source
field alone is not authentication.

Duplicate assertions, within or across reports, MUST be processed
idempotently. Unknown identifiers MUST NOT acknowledge other bundles or
fragments; the recipient MAY ignore such entries without invalidating
otherwise useful entries in the validated record. Where locally tracked
subject information is available, an entry inconsistent with that
subject's fragment identity or required timestamp presence MUST NOT be
used to acknowledge it. Identifiers name bundles or fragments, not
individual retransmission attempts: a delayed or replayed assertion
cannot establish which attempt was received, or that the reporting node
still retains the bundle.

## Interaction with Fragmentation {#fragmentation}

If a node fragments a bundle containing a Hop Reception Report Request
block, the block is replicated in each fragment when bit 0 of its block
processing control flags is set, as recommended in {{flags}}. Each honored
fragment request creates an entry carrying that received fragment's
identity, including its offset and payload length. Multiple such entries
MAY be batched together if they meet {{batching}}, even if their fragments
are subsequently reassembled.

The node MUST capture the requester and fragment identity before
reassembly or replacement or removal of hop-specific blocks, as required
in {{receiving}}. Reassembly MUST NOT substitute a non-fragment identifier
for a bundle that was not received in that form. A report acknowledging
one fragment does not acknowledge the entire original bundle, and
fragments received from different requesting nodes cannot be combined in
a batch addressed to just one of them.

# Relationship to Other Mechanisms

## Bundle Status Reports {#status-reports}

This document does not alter the Bundle Status Report format or the
conditions under which the primary block's status report request flags
cause reports to be generated. Ordinary source-requested reporting remains
unchanged and independent of Hop Reception Reports. The two mechanisms
use different administrative record types, even when their destinations
coincide. An ordinary Bundle Status Report does not satisfy a hop request,
and a Hop Reception Report does not satisfy a source-requested status
report.

The traffic concerns of {{!RFC9171, Section 5.1}} also apply here. Hop
Reception Report generation MUST be disabled by default and enabled only
when the risk of excessive traffic is deemed acceptable. Each honored
request creates a pending entry, not necessarily a separate report
bundle. Batching amortizes carrier overhead across entries, but requests
still consume processing, storage, and return-path capacity. The
forwarding node controls when it requests reports, while the reporting
node controls whether to honor requests and how to aggregate and schedule
reports. Return traffic need not use the immediate link.

## Custody Transfer

This document does not define custody transfer. It provides a forwarding
node with a positive indication that the next hop's BPA has received a
bundle, nothing more. The report does not signify that the receiving node
has undertaken to forward, deliver, or retain the bundle, and the receiving
node incurs no responsibility toward the bundle beyond that which it would
have under {{!RFC9171}} absent this block. What the forwarding node does
with the indication is local policy.

## Convergence Layer Completion Indications

Convergence layer specifications typically define a "transmission success"
indication to the BPA. Hop Reception Reports complement such indications,
not replace them, by providing an explicit BPA-level reception assertion
independent of the convergence layer in use.

Even when the convergence layer indicates success, absence of a report
does not distinguish non-support from transfer failure. Silence can also
result from policy refusal, discarded reporting state, pending
aggregation, an unavailable return contact, receiver failure, or report
loss. Neither silence nor omission of an entry is a negative assertion
about reception.

# Examples {#examples}

These examples show the administrative-record wrapper in CBOR diagnostic
notation, with the symbolic token `TBD2` standing for the unsigned integer
record type code to be allocated by IANA. It is not a CBOR text string.
Each example is a separate record carried in a bundle sourced by the
reporting node `dtn://receiver/` and destined to the validated requester
`dtn://forwarder/`. These two identities do not appear in the entries.
All subject requests in each example came from that requester and have
compatible security treatment.

## One Unfragmented Bundle

The subject source is `dtn://source-a/`, its creation timestamp is
`[1000, 0]`, and it did not request status time:

~~~ cbor-diag
[TBD2, [
  [[[1, "//source-a/"], [1000, 0]]]
]]
~~~

## Different Sources and Mixed Timestamp Presence

These unfragmented bundles have different sources and creation
timestamps. Only the first subject requested status time, so only its
entry contains a reception time (3000 milliseconds after the DTN epoch):

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

# Security Considerations {#security}

## Reflection

A mechanism that causes a node to send a message to a third party on
request is a potential reflection vector: an attacker able to inject a
bundle carrying a Hop Reception Report Request block and a Previous Node
block naming a victim could cause the receiving node to send a report to
the victim.

The validity condition in {{receiving}}, that the Previous Node block's node
ID must identify the convergence-layer peer from which the bundle was
actually received, is the primary mitigation. Over convergence layers that
authenticate the peer node ID, such as TCPCLv4 {{?RFC9174}} with
certificate-based node identification, that authentication can establish
the association. Otherwise, trusted configuration (for example, a
configured peer identity on a controlled link) MUST establish it. A node
MUST NOT honor the request if neither method establishes the association.
Request authentication does not waive the actual-peer association check,
and local policy MAY decline even a valid request.

Addressing a report to the previous hop does not confine its route or
reflection costs to the immediate link. Return routing is local policy,
and authorization, rate limits, and resource bounds remain necessary even
for authenticated requesting peers.

## Integrity and Reporter Authentication

The Hop Reception Report Request block and the Previous Node block are
inserted by the forwarding node and can be integrity-protected by that
node using a Block Integrity Block {{!RFC9172}} with a security source of
the forwarding node. A receiving node with a security association with the
forwarding node can thereby verify that the request was placed by that
node. A source-applied Block Integrity Block cannot cover these blocks,
since they did not exist at the source. Removal or replacement of
hop-specific blocks needs to respect applicable BPSec processing rules.

Authenticating the request does not authenticate the response. A forged or
modified report could induce premature deletion of a retained copy or
suppression of needed retransmission. Before making consequential
retention or retransmission decisions, the report recipient MUST establish
the reporting node's identity and the integrity of all entries it uses,
including their binding to that identity. Protection can use BPSec with
an appropriate security context and trust policy covering the report
payload and binding the carrier's source identity. Security requirements
for all entries must be compatible before batching ({{batching}}).

An authenticated and integrity-protected direct convergence-layer
exchange can provide this assurance when the authenticated peer is the
reporting node and the exchange protects the report. On a multihop return
path, authenticating the last transport peer does not authenticate the
original reporter or protect the entries against modification by earlier
nodes. Such paths require protection and trust arrangements that establish
the original reporter's identity and the integrity of its assertions;
the carrier source field alone is insufficient. When bundle-layer
security services are needed, they SHALL use BPSec as required by
{{!RFC9171, Section 8}}. Authentication establishes who made an assertion,
not that a dishonest reporter actually received or retained the subject.

## Replay and Duplicates

A valid report can be duplicated or replayed. Recipients MUST apply
assertions idempotently and correlate them with locally tracked bundle or
fragment identities and the reporting neighbor ({{report-processing}}).
Neither a bundle identifier nor an optional reception time identifies an
individual retransmission attempt. A replayed report is not evidence of a
new reception or continued retention. Local policy needs to account for
stale reports, bounded correlation state, and the limitations of clocks
when deciding whether an assertion is useful. This specification adds no
sequence-number or anti-replay exchange.

## Resource Exhaustion and Correlated Loss

Requests can consume pending-state storage, encoding work, security
processing, and return-path capacity, even when peers are authenticated.
Large received batches also consume validation and correlation resources.
Nodes SHOULD apply per-peer rate limits and resource budgets in addition
to the configurable limits in {{batching}}. Policy refusal and discarding
pending state are preferable to allowing reporting to exhaust resources
needed for subject-bundle processing. A request does not guarantee a
report, and a refusal is not encoded as a negative entry.

Loss or suppression of one batched report removes multiple assertions at
once. This correlated loss can cause bursts of unnecessary
retransmission if local policy reacts to silence. Batch-size and delay
limits, report scheduling, and retransmission policies should account for
this tradeoff rather than treating each missing entry as an independent
transfer failure.

## Traffic Analysis

Reports expose bundle identifiers and, when requested, reception times.
A batch can link traffic from different sources through a common
forwarding and reporting relationship; its size, membership, and timing
can reveal traffic volumes and contact patterns. Aggregation also means
that report transmission timing is not simply the reception event time.

Convergence-layer confidentiality, such as TCPCLv4 with TLS, protects
content on the links where it is used, but does not hide all traffic
metadata or protect content from intermediate nodes on a multihop return
path. BPSec payload confidentiality can protect entries across that path,
while carrier addressing, sizes, and timing may remain observable.
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

This block was motivated by discussion of the completion semantics of the
QUIC Bundle Protocol Convergence Layer, and the observation that no
convergence layer can, on its own, assert that a peer BPA has received a
bundle.
