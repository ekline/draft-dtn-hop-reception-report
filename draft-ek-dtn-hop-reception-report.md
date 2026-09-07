---
title: "Bundle Protocol Hop Reception Report Request Block"
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
 - status report
 - extension block
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
the bundle's Previous Node block, and the report is an ordinary Bundle
Status Report. No new administrative record types are defined.

--- middle

# Introduction {#intro}

When a Bundle Protocol version 7 (BPv7) {{!RFC9171}} node forwards a bundle
to a neighboring node, the Convergence Layer Adapter (CLA) it uses can
typically indicate that the transfer completed. That indication, however,
reflects only what the convergence layer or underlying transport can
observe: that the data reached the peer's convergence layer (or, in some
cases, only the peer's transport stack). It does not indicate that the
peer's Bundle Protocol Agent (BPA) has received the bundle and accepted it
for processing. If the peer's convergence layer or BPA fails between
transport-level receipt and BPA processing, the bundle may be lost while
the forwarding node believes it was successfully transferred.

BPv7 provides Bundle Status Reports ({{!RFC9171, Section 6.1.1}}) as a
mechanism for nodes to report the reception, forwarding, delivery, or
deletion of a bundle. However, whether reports are requested, and where
they are sent, is determined solely by the bundle's source via the primary
block, which is immutable after creation. A source that requests reception
reports receives one from every node along the path, and a forwarding node
cannot request a report for its own purposes at all.

This document defines the Hop Reception Report Request block, an extension
block that a forwarding node inserts to request that the next-hop node
send a reception report back to the forwarding node. The block carries no
addressing information of its own: the report is sent to the node
identified in the bundle's Previous Node block ({{!RFC9171, Section
4.4.1}}), which the forwarding node inserts as a matter of course. The
report is an ordinary Bundle Status Report with the "reporting node
received bundle" status asserted. The block is removed by the receiving
node and so never travels more than one hop.

The resulting signal, "the next-hop BPA has received this bundle", has
exactly the meaning of the "reporting node received bundle" status
assertion of {{!RFC9171, Section 6.1.1}}: the bundle completed the
reception processing of {{!RFC9171, Section 5.6}} at the receiving node. It
says nothing about what the receiving node will subsequently do with the
bundle. It is nonetheless strictly stronger than any convergence-layer
completion indication and is independent of which convergence layer is in
use. How a forwarding BPA uses this signal is a matter of local policy and
is outside the scope of this document. In particular, this document does
not define a custody transfer mechanism, and the report does not transfer
any responsibility for the bundle between nodes.

## Open Issues {#open-issues}

RFC EDITOR: Please remove this section before publication.

The following points are noted for working group discussion:

Reception timestamp:
: Whether a Bundle Status Report's status item includes a timestamp is
  controlled solely by the "Report status time" flag in the subject bundle's
  primary block, which only the source can set. The forwarding node that
  requests a hop reception report therefore cannot request a timestamp.
  Options include leaving this as-is (no change to the Bundle Status Report
  rules), or defining a flag in this block's flags field that also causes
  the reception time to be included.

Reason code:
: This document uses reason code 0 ("No additional information"). A
  dedicated reason code would let the recipient of a report distinguish a
  hop reception report from a source-requested reception report when both
  are addressed to the same endpoint.

Peer identity check:
: {{receiving}} makes correspondence between the Previous Node block's node
  ID and the convergence-layer peer a hard validity requirement (MUST).
  This means the request is never honored over an unauthenticated
  convergence layer with no configured peer identity. An alternative is a
  SHOULD, with the reflection risk addressed only in
  {{security}}.

Reports after reassembly:
: {{fragmentation}} permits a node that reassembles fragments before
  processing extension blocks to send a single report for the reassembled
  bundle rather than one per fragment. Whether to require per-fragment
  reports instead is open.

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

## Recommended Block Processing Control Flags {#flags}

A forwarding node inserting the Hop Reception Report Request block SHOULD
set the block processing control flags ({{!RFC9171, Section 4.2.4}}) as
follows:

- "Block must be replicated in every fragment" (bit 0): set to 1, so that
  a node receiving only a fragment reports reception of that fragment.
  Bundle Status Reports carry fragment offset and length, so reports for
  distinct fragments are distinguishable.
- "Transmit status report if block can't be processed" (bit 1): set to 0.
- "Delete bundle if block can't be processed" (bit 2): set to 0. Inability
  to honor the request MUST NOT cause the bundle to be lost.
- "Discard block if it can't be processed" (bit 4): set to 1, so that a
  node that does not implement this specification removes the block
  before forwarding the bundle further, preserving the one-hop scope.

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

Upon receiving a bundle containing a Hop Reception Report Request block, a
node that implements this specification SHALL process the block as part of
bundle reception ({{!RFC9171, Section 5.6}}), at the point at which a
source-requested reception status report would be generated, and before any
forwarding decision is made.

The node SHALL determine whether the request is valid. The request is
valid if all of the following hold:

1. The bundle contains exactly one Previous Node block.
2. The node ID in the Previous Node block identifies the convergence-layer
   peer from which this bundle was received. How this correspondence is
   established depends on the convergence layer; see {{security}}.
3. Local policy permits generating the report.

If the request is valid, the node SHALL generate a Bundle Status Report
({{!RFC9171, Section 6.1.1}}) for the bundle with the "reporting node
received bundle" status indicator asserted and the other status indicators
not asserted, with reason code 0 ("No additional information"), and
transmit it to the node ID carried in the Previous Node block. If the
subject bundle's "Report status time" bundle processing control flag is set,
the status item SHALL include the time of reception.

If the request is not valid, the node SHALL treat the block as one that
cannot be processed, and SHALL act according to the block's processing
control flags ({{!RFC9171, Section 5.6}}).

In either case, the node SHALL remove the Hop Reception Report Request
block from the bundle before forwarding it. A node that implements this
specification MUST remove the block regardless of the setting of the
"Discard block if it can't be processed" flag.

If the bundle's primary block also requests reception status reports and
the bundle's report-to endpoint ID is the same as the node ID in the
Previous Node block, a single Bundle Status Report satisfies both requests.

## Interaction with Fragmentation {#fragmentation}

If a node fragments a bundle containing a Hop Reception Report Request
block, the block is replicated in each fragment when bit 0 of its block
processing control flags is set, as recommended in {{flags}}. Each fragment
received then results in a separate reception report identifying the
fragment by offset and length.

If a receiving node reassembles fragments before processing extension
blocks, it MAY instead generate a single reception report for the
reassembled bundle. In this case the report SHALL NOT include fragment
offset and length.

# Relationship to Other Mechanisms

## Bundle Status Reports

This document does not alter the Bundle Status Report format or the
conditions under which the primary block's status report request flags
cause reports to be generated. It adds one additional condition under which
a reception report is generated, with an alternative destination.

The traffic considerations of {{!RFC9171, Section 5.1}} apply. Each honored
request generates one report bundle; a forwarding node that inserts this
block into every bundle it forwards should expect a corresponding return
flow. Unlike source-requested reception reports, the traffic is confined to
a single hop and returns over the same neighbor relationship, and the
forwarding node has direct control over when it is incurred.

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
indication to the BPA. The Hop Reception Report Request block is intended
to complement such indications, not replace them. A forwarding node
receiving a convergence layer success indication but no reception report
can distinguish "the next hop does not implement or declined this request"
from "the transfer failed", since the former still yields the convergence
layer indication.

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
certificate-based node identification, this check is strong. Over
convergence layers that do not authenticate the peer, a receiving node
SHOULD NOT honor the request unless the peer's node ID is known by other
means (for example, static configuration of the link), and MAY decline to
honor it at all.

Because the report is addressed to the previous hop, which is by definition
a directly connected neighbor, the amplification potential is in any case
limited to the immediate link.

## Integrity

The Hop Reception Report Request block and the Previous Node block are
inserted by the forwarding node and can be integrity-protected by that
node using a Block Integrity Block {{?RFC9172}} with a security source of
the forwarding node. A receiving node with a security association with the
forwarding node can thereby verify that the request was placed by that
node. A source-applied Block Integrity Block cannot cover these blocks,
since they did not exist at the source.

## Traffic Analysis

Reception reports reveal to an observer of the link that a bundle was
received by the receiving BPA, and their timing may reveal processing
latency. Convergence layers providing confidentiality, such as TCPCLv4 with
TLS, conceal this from off-link observers.

# IANA Considerations {#iana}

IANA is requested to assign a value from the "Bundle Block Types"
subregistry of the "Bundle Protocol" registry, in the range for Bundle
Protocol Version 7, as follows:

| Bundle Protocol Version | Value | Description | Reference |
|-------------------------|-------|-------------|-----------|
| 7 | TBD1 | Hop Reception Report Request | This document |
{: #tab-block-type align="left" title="Bundle Block Type Registration"}

--- back

# Acknowledgments
{:numbered="false"}

This block was motivated by discussion of the completion semantics of the
QUIC Bundle Protocol Convergence Layer, and the observation that no
convergence layer can, on its own, assert that a peer BPA has received a
bundle.
