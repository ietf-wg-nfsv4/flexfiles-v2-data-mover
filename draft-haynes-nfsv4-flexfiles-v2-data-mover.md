---
title: Proxy-Driven Data Mover for Flexible Files Version 2
abbrev: FFv2 Data Mover
docname: draft-haynes-nfsv4-flexfiles-v2-data-mover-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: General
workgroup: Network File System Version 4
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC7861:
  RFC7862:
  RFC7863:
  RFC8178:
  RFC8881:
  RFC9289:
  I-D.haynes-nfsv4-flexfiles-v2:

informative:
  RFC1813:
  RFC8435:

--- abstract

Parallel NFS (pNFS) with the Flexible Files Version 2 layout
type supports client-side erasure coding and per-chunk repair
between clients and data servers.  This document extends that
architecture with a proxy server (PS) role: a registered peer
of the metadata server that the metadata server directs, via
callback operations on a dedicated session, to move a file
from one layout to another or to reconstruct a whole file
from surviving shards.  The same mechanism provides codec
translation for clients that cannot participate in a file's
native erasure code, including NFSv3 clients.

--- note_Note_to_Readers

Discussion of this draft takes place on the NFSv4 working group
mailing list (nfsv4@ietf.org), which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=nfsv4).
Source code and issues list for this draft can be found at
[](https://github.com/ietf-wg-nfsv4/flexfiles-v2-data-mover).

Working Group information can be found at
[](https://github.com/ietf-wg-nfsv4).

--- middle

# Introduction

The Flexible Files Version 2 layout type
({{I-D.haynes-nfsv4-flexfiles-v2}}) introduces client-side
erasure coding for pNFS and a per-chunk repair protocol
(CB_CHUNK_REPAIR) that lets the metadata server direct an
active client to reconstruct individual damaged chunks.  That
mechanism is sufficient for repairs whose scope is a handful of
chunks in a file that has at least one live client.

Three classes of work are outside the per-chunk repair model.
The first is **whole-file repair**: the case in which enough
data servers have failed that per-chunk reconstruction would
require visiting every chunk, or in which no live client is
available to drive the repair at all.  The second is **layout
transitions**: a file must move from one layout geometry to
another for policy reasons (migrating to a new coding type, or
re-mirroring), for maintenance reasons (evacuating a data
server ahead of decommission), or for environmental reasons
(moving between transport-security profiles or between
filehandle backends).  The third is **codec translation**: a
client that cannot participate in the file's native codec --
including every NFSv3 client, and any legacy or minimal NFSv4
client that does not implement the file's encoding type --
still needs to read and write the file.

This document specifies a **proxy server (PS)** role to
address those three cases with a single mechanism: the PS
opens a session to the metadata server and registers its
capabilities; the MDS issues callback operations on the
session's back-channel to direct the PS to move, repair,
poll, or cancel; the PS reports progress on the fore-channel.
Clients discover the PS through the normal pNFS layout path
-- a layout that names the PS with a new data-server flag
(FFV2_DS_FLAGS_PROXY) -- and route their I/O through it while
it is active.

This design codifies a mechanism that is common, but
implementation-specific, in existing Flex Files v1 deployments.
Today the lack of a standardized version is the single biggest
interop gap between pNFS and parallel-filesystem competitors
that already expose migration primitives.

# Requirements Language

{::boilerplate bcp14-tagged}

## Relation to the Main Draft

{{I-D.haynes-nfsv4-flexfiles-v2}} defines CB_CHUNK_REPAIR and
the per-chunk repair model.  This document is the companion
whole-file and per-client mechanism.  Per-chunk and whole-file
operations are mutually exclusive for a given file at a given
time; coexistence rules are in {{interaction}}.

# Scope

This section draws the boundary between the wire-level
mechanism defined here and the much larger space of useful
behaviours a proxy implementation might support.  Drawing
the boundary tightly keeps the protocol small enough to
specify and implement in a single revision; everything
beyond the boundary is either future work or implementation
latitude, and both categories are called out below so later
readers know which is which.

## In Scope

This document defines a new protocol role, a new session,
and a small set of operations that flow on that session.
Around that core it specifies the layout conventions a
client observes while a proxy operation is active, the
rules that match a client to a co-resident proxy, the
credential-forwarding rules a translating proxy must
follow, and the recovery semantics for the three actor
failures that matter during an operation (PS, MDS, DS).

The new role is the **proxy server (PS)**, distinct from
the MDS and DS roles defined in
{{I-D.haynes-nfsv4-flexfiles-v2}}.  A PS registers with an
MDS and, on receipt of a directive from that MDS, performs
a move, a repair, or an ongoing codec translation on behalf
of a client.  The PS is opaque to most clients and is
visible to the MDS through a dedicated session that the PS
itself opens: the fore-channel (PS to MDS) carries
PS-initiated ops, and the back-channel (MDS to PS) carries
the MDS-initiated directives that drive work.

The fore-channel surface is deliberately small.
PROXY_REGISTRATION ({{sec-PROXY_REGISTRATION}}) lets the PS
declare the codec set it supports, its co-residency
affinity token, and its lease.  PROXY_PROGRESS
({{sec-PROXY_PROGRESS}}) lets the PS report interim and
terminal status of an in-flight operation.  The
back-channel carries the four directives that together
cover the entire move, repair, and translation lifecycle:
CB_PROXY_MOVE ({{sec-CB_PROXY_MOVE}}) directs the PS to
move a file from a source layout to a destination layout;
CB_PROXY_REPAIR ({{sec-CB_PROXY_REPAIR}}) specialises that
directive for the case where the source layout is degraded;
CB_PROXY_STATUS ({{sec-CB_PROXY_STATUS}}) lets the MDS poll
current status out-of-band from the PS's own progress
reports; and CB_PROXY_CANCEL ({{sec-CB_PROXY_CANCEL}}) asks
the PS to terminate an in-flight operation before
completion.

Around the operation set, the document specifies the layout
conventions a client sees during a proxy operation and how
a client discovers the PS in the first place.  Co-residency
attestation -- the "affinity matching" machinery that makes
host-local proxy shortcuts safe -- travels as an affinity
token carried in PROXY_REGISTRATION and is described in
{{sec-affinity}}.

Codec translation for codec-ignorant clients, including
NFSv3 clients, is in scope, and is the one case that
stretches the move/repair vocabulary.  The same proxy
machinery that handles move and repair also provides the
persistent per-client translation that lets a client
incapable of participating in a file's native codec still
read and write the file.  Unlike move and repair, which are
transient transitions on the file, codec translation is an
ongoing routing arrangement that persists as long as the
codec-ignorant client is active.

Credential-forwarding rules for a proxy that translates on
behalf of a client are defined in the Security
Considerations section.  The proxy is a translator, not an
authority: authorization decisions MUST remain with the
MDS, using the client's forwarded credentials.  Getting
this boundary wrong turns the PS into a privilege-elevation
vehicle, so the rules are stated normatively and enforced
at the MDS rather than policed on the wire.

Finally, the document defines recovery semantics for the
three actor failures that matter during a proxy operation
-- PS failure, MDS failure, and DS failure -- each with its
own fencing and re-registration rules so that a
mid-operation crash does not leave a file in an
unrecoverable state.

## Out of Scope {#sec-scope-out}

The items below are deliberately deferred.  They split
into three groups: features whose absence was an explicit
design decision (delta journaling, partial-range moves),
orchestration that belongs to a layer above a single
proxy (multi-proxy pipelines, automated load balancing),
and proxy-internal behaviour that does not surface on the
wire and therefore needs no standardisation.  Nothing on
this list is precluded by the current design; each is a
reasonable future extension.

**Journaling and partial moves.**  CB_PROXY_MOVE in this
revision is always whole-file, and is either quiesced
(clients recalled and a new layout issued after the move
completes) or dual-written (the PS replicates writes to
source and destination while active).  Delta-journaling
mechanisms -- capturing writes against an otherwise-offline
source, replaying them on completion, or maintaining
reference integrity across detached clones -- are a future
extension, as is a partial-range CB_PROXY_MOVE that would
move a byte range while the rest of the file stays on the
source.  The quiesced and dual-write modes together cover
every motivating scenario the design currently has.

**Orchestration beyond a single proxy.**  Multi-proxy
pipelines (staged moves for very large files) and
automated load balancing or predictive selection across
registered proxies are out of scope.  An MDS in this
revision selects a single PS per operation; load
distribution across many proxies, when it matters, is
expected to be handled by the MDS's selection policy and
does not surface as new wire protocol.

**Server-side copy as an alternative path.**  Integration
with server-side copy ({{RFC7862}} Section 4) as an
alternative to CB_PROXY_MOVE for single-file moves within
one namespace is adjacent work.  The two mechanisms are
complementary (server-side copy is a client-directed
intra-server operation; CB_PROXY_MOVE is an MDS-directed
inter-server operation), and their intersection -- for
example, using server-side copy under the hood of a
CB_PROXY_MOVE -- is better specified in its own extension
rather than bolted into this document.

**Proxy-internal features that do not surface on the
wire.**  A proxy MAY implement content-integrity and
error-correction layers, encryption and compression
pass-through, log-structured write staging, sector-
alignment normalisation, and POSIX-loopback shortcuts when
proxy and client are co-resident.  These are useful
motivating scenarios for CB_PROXY_MOVE but do not require
new protocol surface beyond what CB_PROXY_MOVE already
provides, and so they are left to implementation rather
than standardised here.

# Use Cases

Seven motivating scenarios converge on the same mechanism.
Six of them are transient transitions on the file itself: the
file is changing state (being ingested, re-coded, evacuated,
reconstructed, migrated between transport-security profiles,
or migrated between filehandle backends), and the transition
is what the PS drives to completion.  The seventh is
qualitatively different: the file is not changing, but
specific clients -- those that cannot participate in the
file's native codec -- are routed through a PS persistently,
for as long as they are active.

The distinction matters for how the MDS schedules work.
Transient transitions have a terminal state; the MDS expects
each one to complete (via terminal PROXY_PROGRESS) and then
to retire the associated layout.  The persistent routing case
has no terminal state for the file as a whole; the PS stays in
the layouts of codec-ignorant clients as long as those
clients are open.

In every case, a registered PS becomes the source of truth
for a file's data during the operation, and clients are
redirected to route I/O through that PS rather than directly
to the original layout's data servers.  The individual
scenarios are described below.

## Administrative Ingest

An administrator rsyncs a file from an external source into the
cluster as a single-copy file.  Server policy requires the file
to be mirrored or erasure coded.  The MDS issues CB_PROXY_MOVE to
a registered proxy whose codec set covers the destination
layout.  The proxy populates the destination from the source,
while any client that opens the file during the move sees a
layout that routes I/O through the proxy.

The source "layout" may not even be a Flex Files layout; it
could be a non-pNFS NFS mount that the proxy reads as an
NFSv4.2 client.  Throughout the move the proxy presents the
file to pNFS clients as if the move had not started, while
populating the destination in the background.

## Policy-Driven Layout Transition

A server-objective or policy change ("files older than 30 days
must be erasure coded", "high-access-rate files must have
additional mirrors") requires transforming a file's layout
without user visibility.  The transformation is purely a layout
change; the file contents are unchanged except at the shard
level.  CB_PROXY_MOVE carries the new layout's geometry and
coding type; the proxy reshapes the file's shards to match.
Because the transformation type (encode / decode / transcode)
is entirely specified by the (source, destination) layout pair,
the op does not need a separate transformation-class field.

## DS Maintenance / Evacuation

A data server is scheduled for maintenance (hardware
replacement, software upgrade, decommission).  All files whose
layouts reference that DS must be evacuated to replacement
DSes before it is taken offline.  CB_PROXY_MOVE is issued against
each file, with source layouts pointing at the outgoing DS and
destination layouts pointing at the replacement.  Evacuation
can be large-scale (thousands or millions of files); running
per-client per-chunk repair over every file would be
prohibitively expensive, but a single registered proxy can
drive many concurrent CB_PROXY_MOVE operations.

## Whole-File Repair

Multiple DSes have failed such that per-chunk repair cannot
reconstruct the file in place.  The MDS constructs a new layout
backed by replacement DSes and issues CB_PROXY_REPAIR to a proxy,
which drives reconstruction from whatever surviving shards
remain.  If fewer than k shards survive across the mirror set,
the operation terminates with NFS4ERR_PAYLOAD_LOST, matching
the per-chunk repair semantics in the Repair Client Selection
section of {{I-D.haynes-nfsv4-flexfiles-v2}}.

## TLS Coverage Transition

A file whose layout currently points at non-TLS-capable DSes
needs to be migrated to TLS-capable DSes, or vice versa (an
inventory change, a policy change mandating transport security,
onboarding a new storage class whose DSes are TLS-only).
CB_PROXY_MOVE applies: the destination DSes have the required
transport security profile, the source DSes are retired.  A
client that arrives mid-transition is routed through the proxy
and does not directly see the heterogeneous DS set.

The proxy establishes its own RPC connections to source and
destination DSes, potentially with different transport security
profiles (non-TLS to source, mutual TLS {{RFC9289}} to
destination, or any other combination).  The proxy's per-DS
security is independent of the client's security to the proxy.

## Filehandle / Storage-Backend Transition

A DS changes the filehandles it issues for a file; this
happens when the DS's underlying storage is migrated (e.g.,
from one backend object store to another) and the old
filehandles become unresolvable on the new backend.  Without a
proxy, every client holding a layout has to be individually
recalled and re-issued.  With a proxy, the MDS points all
clients at the proxy (keeping their existing stateids and FHs
intact), the proxy reconciles old-to-new FHs internally, and
clients are recalled only at the end.

This same mechanism covers several related situations: an
NFSv3-to-NFSv4.2 DS protocol upgrade where the DS FHs change
as a side effect of migrating from {{RFC1813}} to
{{RFC8881}} semantics; a DS-side format change that
invalidates existing FHs (for example, a transition from a
local POSIX store to an object store); and backend-opaque FH
migration where the DS's FH structure is internally
versioned and old clients hold stale versions.

## Codec Translation for Codec-Ignorant Clients {#sec-codec-translation}

The coding-type registry defined in the IANA Considerations of
{{I-D.haynes-nfsv4-flexfiles-v2}} is expected to grow.  Not
every client is required to implement every registered codec;
a minimal client, a legacy client, or an NFSv3 client typically
cannot participate in erasure-coded files at all.  Per the
codec-negotiation rules in {{I-D.haynes-nfsv4-flexfiles-v2}},
such a client either retries with a different supported_types
hint, falls back to MDS-terminated I/O, or (this case) is
routed through a proxy that translates on its behalf.

Unlike the move / repair / evacuation / transition use cases
above, codec translation is **persistent per client**.  The
file itself is not changing state.  What changes is the layout
the MDS hands to a codec-ignorant client: that client gets a
layout with FFV2_DS_FLAGS_PROXY set and a coding_type the
client does support (typically FFV2_CODING_MIRRORED, or for
NFSv3 clients just a flat NFSv3 data surface).  The proxy
encodes and decodes on the fly against the real DSes; the
client sees a flat file.

The same file may be accessed directly by codec-aware clients
(with a normal layout naming the real DSes) and through the
proxy by codec-ignorant clients (with a proxy layout)
simultaneously.  The MDS issues a different layout per
request; FFV2_DS_FLAGS_PROXY is set only in the layouts that
need translation.

### Mechanism

A translating proxy runs two sides that meet internally.  On
its **client-facing** side it speaks the protocol the
codec-ignorant client can speak: for an NFSv3 {{RFC1813}}
client that is an NFSv3 server that re-exports the MDS's
namespace; for a legacy NFSv4.2 client that understands only
some codecs, it is an NFSv4.2 data-server surface presenting
FFV2_CODING_MIRRORED (or an equivalent codec the client
supports).  On its **MDS-facing** side it is an NFSv4.2
client to the MDS plus whatever DS protocol the MDS's real
DSes speak.  The proxy translates each client-facing op into
the corresponding MDS or DS op, applies the codec
transformation between the two, and returns results.

For an NFSv3 client, a read flows:

-  Client: NFSv3 `READ` against the proxy.
-  Proxy: if it does not hold a layout for the file, issues
   `LAYOUTGET` on the MDS with the client's forwarded
   credentials (see Security Considerations).
-  Proxy: issues `CHUNK_READ` (or v3 `READ` if the DS is
   NFSv3) against the real DSes, decodes the shards back to
   plaintext.
-  Proxy: returns the plaintext bytes in the NFSv3 READ
   reply.

A write flows:

-  Client: NFSv3 `WRITE` with stable_how and a byte range.
-  Proxy: encodes the bytes per the file's codec, issues
   `CHUNK_WRITE` / `CHUNK_FINALIZE` / `CHUNK_COMMIT` against
   the real DSes.
-  Proxy: returns NFSv3 WRITE ok with the stable_how it was
   able to honor (which may be downgraded based on the
   back-end DS's own stable_how behavior).

### Why the same PROXY_REGISTRATION machinery

The registered-PS mechanism gives the MDS the information it
needs for translation-proxy selection: `prr_codecs` enumerates
the codecs the PS can translate between, the MDS <-> PS
session (fore-channel and back-channel) carries both
directions of control-plane traffic, and the lease bounds the
relationship.  No new op is required for the translation case
-- the existing `PROXY_REGISTRATION` covers it.  `CB_PROXY_MOVE`
and `CB_PROXY_REPAIR` are not used for pure translation (the
file is not moving or being repaired); the PS simply serves
the codec-ignorant client's I/O requests
against the unchanged source layout.

# Design Model

## Roles

This document introduces a third role alongside the pNFS
metadata server (MDS) and data server (DS):

Proxy server (PS):
:  A persistent, registered peer of the MDS that carries out
   whole-file operations on the MDS's behalf -- moving file
   content between layouts, reconstructing files whose source
   layout has been damaged, and translating codecs on behalf of
   clients that cannot participate in the file's native
   encoding.  A PS is a distinct role from a DS; a given server
   MAY implement both, and typically does, but the protocol
   does not require that.  The MDS sees the PS through a
   dedicated session whose direction is defined in
   {{sec-design-session}}.

The existing roles are unchanged:

Metadata server (MDS):
:  As defined in {{I-D.haynes-nfsv4-flexfiles-v2}}: the
   coordinator for each file, and the authority that issues
   layouts, manages stateids, and selects repair participants.

Data server (DS):
:  As defined in {{I-D.haynes-nfsv4-flexfiles-v2}}: serves the
   CHUNK data path to pNFS clients.

Only one of the three pairs carries new ops in this document.
The MDS <-> PS pair gains the new PS-to-MDS regular ops for
registration and progress ({{sec-new-ops}}) and the new
MDS-to-PS callback ops for move, repair, status, and cancel
({{sec-new-cb-ops}}).  The Client <-> PS pair gains no new
ops: clients reach a PS through the normal pNFS data path,
seeing it as a DS with FFV2_DS_FLAGS_PROXY set in the layout
({{sec-layout-shape}}).  The MDS <-> DS pair is also
unchanged; the tight-coupling control session in
{{I-D.haynes-nfsv4-flexfiles-v2}} carries over as defined
there.

## Session Between MDS and PS {#sec-design-session}

The PS opens a session to the MDS.  On that session the PS
is the session client, so PS-initiated ops
(PROXY_REGISTRATION and PROXY_PROGRESS) flow as regular ops
on the fore-channel.  The MDS is the session server, so
MDS-initiated ops (CB_PROXY_MOVE, CB_PROXY_REPAIR,
CB_PROXY_STATUS, and CB_PROXY_CANCEL) flow as callback ops
on the back-channel of the same session.  A single session
thus carries both directions of traffic.

The session direction is intentionally opposite to the
MDS -> DS tight-coupling control session in
{{I-D.haynes-nfsv4-flexfiles-v2}}: that session is opened by
the MDS to carry MDS-originated stateid management to a DS.
The MDS <-> PS session is opened by the PS because registration
is a PS-initiated act -- the PS is saying "here I am, with
these capabilities."  Without a PS-to-MDS direction the
capability-advertisement would have to be inferred from
session-setup flags alone, which is inadequate for the range
of capabilities a PS can usefully advertise (codec set,
affinity token, and -- as the DEVICEID_REGISTRATION open
question anticipates -- fault-zone coordinates and other
deployment attributes).

A consequence of this direction choice is that a server
that implements both the DS and PS roles toward the same MDS
runs two sessions between the same pair of hosts: the MDS
opens the DS tight-coupling session toward the box, and the
box's PS opens the PS session toward the MDS.  That is two
EXCHANGE_ID exchanges, two CREATE_SESSION exchanges, and two
TCP connections.  The cost is asymmetric: the MDS consumes
no additional reserved local port under AUTH_SYS (the PS
session is inbound from the MDS's point of view), while the
combined DS+PS box consumes one additional outbound reserved
port per registered MDS (the PS session is outbound from the
box).  In deployments that use RPCSEC_GSS ({{RFC7861}}) or
RPC-over-TLS ({{RFC9289}}) on the PS session --
which the credential-forwarding rules in
{{sec-security}} recommend for any PS that translates on
behalf of clients -- reserved-port trust is not in use and
this cost does not apply.  In a strict AUTH_SYS-only
deployment the additional reserved port is a real but
typically negligible cost, because a storage box's outbound
NFS traffic is usually limited to one connection per MDS it
is registered with.  Consolidating the two sessions is
possible but would require flipping the channel on which
either the tight-coupling ops or the proxy ops travel, with
the corresponding ripple in {{I-D.haynes-nfsv4-flexfiles-v2}};
that consolidation is not attempted here.

## Flow Summary

The PS opens a session to the MDS and issues
PROXY_REGISTRATION, declaring its supported codecs and its
affinity token; the MDS records the registration and returns
a registration id with a granted lease.  When the MDS decides
to move or repair a file, it selects a registered PS whose
capabilities match the operation and issues CB_PROXY_MOVE or
CB_PROXY_REPAIR on the back-channel of that PS's session.
The directive carries the full specification of the work --
source layout, destination layout, and scheduling parameters.
The PS drives the operation to completion and reports
terminal status (and, optionally, periodic progress) by
issuing PROXY_PROGRESS on the fore-channel of the same
session.  The MDS MAY at any time poll the PS for current
status via CB_PROXY_STATUS, or request cancellation via
CB_PROXY_CANCEL.

Clients interact with the PS through the normal layout path.
During a proxy operation the MDS hands out layouts that name
the PS as a DS entry with FFV2_DS_FLAGS_PROXY set; clients
route CHUNK I/O to that entry.  Clients that arrive
mid-operation see the proxy layout from the start and need
no additional signalling; clients that held an older (non-
proxy) layout are recalled via CB_LAYOUTRECALL and reacquire.

## Message Sequence: Policy-Driven Move

The simplest flow -- a quiesced whole-file move for a policy
transition.  Shown as a wire-level message sequence between
the three protocol actors; clients are elided because in the
quiesced case they are recalled before the PS work starts.

~~~
  PS                                MDS
  |                                 |
  | ---- CREATE_SESSION ----------> | (PS opens session to MDS)
  | <--- session est. ------------- |
  |                                 |
  | ---- PROXY_REGISTRATION ------> | (advertise codecs, affinity)
  | <--- reg_id, granted_lease ---- |
  |                                 |
  |                                 | <-- (policy decides: move)
  |                                 |
  | <---- CB_PROXY_MOVE ----------- | (back-channel)
  | ----- operation_id -----------> |
  |                                 |
  |  [PS drives move: reads source  |
  |   DSes, encodes per destination |
  |   codec, writes destination     |
  |   DSes]                         |
  |                                 |
  | ---- PROXY_PROGRESS ----------> | (interim, bytes_done > 0)
  | <--- NFS4_OK ------------------ |
  |                                 |
  |  ...                            |
  |                                 |
  | ---- PROXY_PROGRESS ----------> | (terminal, ppa_status=
  |                                 |  NFS4_OK, ppa_terminal=true)
  | <--- NFS4_OK ------------------ |
  |                                 |
  |                                 | --- CB_LAYOUTRECALL --->
  |                                 |     (to affected clients)
  |                                 |
  |                                 | <-- LAYOUTRETURN ------
  |                                 |     (from each client)
  |                                 |
  |                                 | (MDS retires source DSes
  |                                 |  and issues new layout
  |                                 |  naming destination DSes)
  v                                 v
~~~
{: #fig-seq-policy-move title="Message sequence for a policy-driven move"}

## Message Sequence: Whole-File Repair

Same shape as a move, but the source layout is degraded and
the MDS issues CB_PROXY_REPAIR.  Terminal outcomes:

-  **NFS4_OK**: the PS reconstructed the file; the MDS
   proceeds as in {{fig-seq-policy-move}}.
-  **NFS4ERR_PAYLOAD_LOST**: fewer than k shards survived
   across the mirror set; the PS sends a terminal
   PROXY_PROGRESS with this status, and the MDS marks the
   affected byte ranges lost.  No CB_LAYOUTRECALL is issued
   because there is no valid destination layout to issue.

## Message Sequence: MDS-Initiated Cancellation

~~~
  PS                                MDS
  |                                 |
  |  [in-flight CB_PROXY_MOVE]      |
  |                                 |
  |                                 | <-- (cancel decision)
  | <---- CB_PROXY_CANCEL --------- |
  | ----- NFS4_OK ----------------> |
  |                                 |
  |  [PS stops work, releases       |
  |   resources]                    |
  |                                 |
  | ---- PROXY_PROGRESS ----------> | (terminal, ppa_status=
  |                                 |  implementation-chosen
  |                                 |  non-OK code)
  | <--- NFS4_OK ------------------ |
  v                                 v
~~~
{: #fig-seq-cancel title="Message sequence for MDS-initiated cancellation"}

# New NFSv4.2 Operations {#sec-new-ops}

This document defines two new NFSv4.2 operations that a proxy
server (PS) issues to the metadata server (MDS) on the
fore-channel of the PS -> MDS session defined in
{{sec-design-session}}.  PROXY_REGISTRATION (91) is issued
once at session setup and on renewal.  PROXY_PROGRESS (92) is
issued by the PS to report terminal and periodic progress for
in-flight move or repair operations.  Neither operation is
sent by pNFS clients.

The MDS-initiated ops -- CB_PROXY_MOVE, CB_PROXY_REPAIR,
CB_PROXY_STATUS, and CB_PROXY_CANCEL -- are callback ops on
the back-channel of the same session and are defined in
{{sec-new-cb-ops}}.

~~~ xdr
/// /* New operations for the Data Mover (PS -> MDS) */
///
/// OP_PROXY_REGISTRATION   = 93;
/// OP_PROXY_PROGRESS       = 94;
~~~
{: #fig-data-mover-opnums title="Data Mover operation numbers"}

Opcodes 91 and 92 are chosen to continue the MDS-to-DS
control-plane range that {{I-D.haynes-nfsv4-flexfiles-v2}}
opens at 88 (TRUST_STATEID through BULK_REVOKE_STATEID at
88-90).  If other in-flight NFSv4.2 extensions collide on
these values during IANA coordination, the final assignment
will be reconciled by the consuming RFC editor.

The following amendment blocks extend the nfs_argop4 and
nfs_resop4 dispatch unions from {{RFC7863}} with the new ops.
A consumer that combines this document's extracted XDR with the
RFC 7863 XDR applies the amendments at the unions' extension
point.

~~~ xdr
/// /* nfs_argop4 amendment block */
///
/// case OP_PROXY_REGISTRATION:
///     PROXY_REGISTRATION4args opproxyregistration;
/// case OP_PROXY_PROGRESS:
///     PROXY_PROGRESS4args opproxyprogress;
~~~
{: #fig-nfs_argop4-amend title="nfs_argop4 amendment block"}

~~~ xdr
/// /* nfs_resop4 amendment block */
///
/// case OP_PROXY_REGISTRATION:
///     PROXY_REGISTRATION4res opproxyregistration;
/// case OP_PROXY_PROGRESS:
///     PROXY_PROGRESS4res opproxyprogress;
~~~
{: #fig-nfs_resop4-amend title="nfs_resop4 amendment block"}

## Operation 93: PROXY_REGISTRATION - Register as Data Mover {#sec-PROXY_REGISTRATION}

### ARGUMENTS

~~~ xdr
/// struct PROXY_REGISTRATION4args {
///     ffv2_coding_type4  prr_codecs<>;
///     opaque             prr_affinity<>;
///     uint32_t           prr_lease;
///     uint32_t           prr_flags;
/// };
~~~
{: #fig-PROXY_REGISTRATION4args title="XDR for PROXY_REGISTRATION4args"}

### RESULTS

~~~ xdr
/// struct PROXY_REGISTRATION4resok {
///     uint64_t           prr_registration_id;
///     uint32_t           prr_granted_lease;
/// };
///
/// union PROXY_REGISTRATION4res switch (nfsstat4 prr_status) {
///     case NFS4_OK:
///         PROXY_REGISTRATION4resok  prr_resok4;
///     default:
///         void;
/// };
~~~
{: #fig-PROXY_REGISTRATION4res title="XDR for PROXY_REGISTRATION4res"}

### DESCRIPTION

A proxy server (PS) calls PROXY_REGISTRATION on the
fore-channel of its session to the MDS
({{sec-design-session}}) to declare its capabilities.  The
MDS records the registration and MAY select that PS for
subsequent CB_PROXY_MOVE and CB_PROXY_REPAIR directives on
the session's back-channel.

The prr_codecs field lists the ffv2_coding_type4 values the
PS supports.  The PS MUST be able to encode, decode, and
transcode between any pair of values in this list.  Because
the transformation class of a CB_PROXY_MOVE is inherent in
the (source, destination) layout pair, this codec-set
membership is all the capability information the MDS needs
to match.  An empty list results in NFS4ERR_INVAL in this
revision.

The prr_affinity field is an opaque token the PS supplies for
co-residency attestation with a client.  The MDS does not
interpret it.  At layout-grant time the MDS compares
prr_affinity against the requesting client's co_ownerid
(Section 18.35 of {{RFC8881}}); a match indicates that the
client and the PS share a host, and the MDS MAY use it as
input to PS selection.  See {{sec-affinity}} for the matching
algorithm.  An empty prr_affinity means the PS does not claim
co-residency with any client.

The prr_lease field is the lease duration the PS requests in
seconds.  The MDS MAY grant a shorter one, returned in
prr_granted_lease.  The PS MUST renew before the granted
lease expires; on expiry the MDS drops the registration and
any in-flight CB_PROXY_MOVE or CB_PROXY_REPAIR is reassigned.

The prr_flags field is reserved for future use.  In this
revision the PS MUST set prr_flags to 0, and an MDS that
receives a PROXY_REGISTRATION with any bit of prr_flags set
MUST reject it with NFS4ERR_INVAL.

The "reject, don't ignore" rule follows the NFSv4 extension
model in {{RFC8178}}.  Section 8 of {{RFC8178}} specifies
that when a flag bit is used that is not known in the
specified minor version, NFS4ERR_INVAL is returned; Section
4.4.3 of {{RFC8178}} then explains that this same error is
how a requester determines whether the responder understands
the bit.  Silently ignoring an unknown bit would break that
discovery contract: a PS that sets a future capability bit
against an MDS that pre-dates the bit could not tell whether
the MDS honored the capability or simply dropped it.

A future revision of this specification (or a successor
document that updates it) MAY define new bit values in
prr_flags, following the extension rules of Section 4.2 of
{{RFC8178}}.  A PS that understands a newly defined bit MAY
set it when registering with an MDS that supports it; on
NFS4ERR_INVAL the PS MAY retry with the bit cleared,
treating the response as the {{RFC8178}} Section 4.4.3
signal that the MDS does not recognize the bit.

On success, the MDS returns a prr_registration_id that
identifies this registration.  The PS uses it to correlate
CB_PROXY_MOVE and CB_PROXY_REPAIR directives the MDS
subsequently issues, and to renew the registration before
the granted lease expires.

Registration conveys capabilities only; the PS's network
endpoint is conveyed through the same deviceinfo channel as
any other DS's address.  When the MDS selects a PS for an
operation, the layout issued to clients includes a
ffv2_data_server4 entry pointing at the PS's existing
deviceinfo.

PROXY_REGISTRATION is issued on the fore-channel of the
MDS <-> PS session.  That session is opened by the PS, not by
the MDS; it is distinct from the MDS -> DS tight-coupling
control session defined by {{I-D.haynes-nfsv4-flexfiles-v2}}
even when the same host acts as both DS and PS.  The PS MUST
present EXCHGID4_FLAG_USE_NON_PNFS on the session so that
the MDS can distinguish it from a regular pNFS client.  An MDS
that receives PROXY_REGISTRATION on a session whose owning
client did not present EXCHGID4_FLAG_USE_NON_PNFS MUST reject
it with NFS4ERR_PERM.

Before recording the registration, the MDS MUST validate the
PS's transport-security identity against a deployment
allowlist.  Acceptable identities are the PS's RPCSEC_GSS
machine principal or the PS's RPC-over-TLS client-certificate
identity, matching the MDS <-> PS transport-security rule in
{{sec-credential-forwarding}}.  The MDS MUST reject a
PROXY_REGISTRATION from any identity not on the allowlist with
NFS4ERR_PERM.  AUTH_SYS, even over a transport that is
otherwise authenticated, is never a valid authenticator for
this operation; the MDS MUST reject it.

Because one PS proxies one MDS, a successful rogue registration
displaces the legit PS and returns NFS4ERR_STALE to every
client holding cached filehandles against the previous PS.  To
guard against registration squatting, the MDS MUST refuse a
new PROXY_REGISTRATION from an allowlisted identity while an
existing registration from that same identity still holds a
valid lease; the MDS returns NFS4ERR_DELAY and SHOULD log the
conflict.  A renewal -- distinguished by the PS re-presenting
the same prr_registration_id it received on the prior
registration -- is not squatting and the MDS MUST accept it
(refreshing the granted lease).

Registration revocation before lease expiry is not a dedicated
operation in this revision.  An MDS that needs to revoke a PS
before its lease expires MUST cease issuing CB_PROXY_MOVE,
CB_PROXY_REPAIR, CB_PROXY_STATUS, and CB_PROXY_CANCEL to that
PS; MUST return NFS4ERR_STALE_CLIENTID on subsequent
PROXY_PROGRESS or PROXY_REGISTRATION-renewal from the revoked
PS; and MUST handle in-flight CB_PROXY_MOVE or CB_PROXY_REPAIR
as if the lease had expired (see the lease-expiry paragraph
above): preferably CB_PROXY_CANCEL where the back-channel is
still reachable, layout revert to the pre-operation state
otherwise.  A future revision MAY define a dedicated
PROXY_REVOKE operation if operational experience shows lease
revocation through silence is insufficient.

## Operation 94: PROXY_PROGRESS - Report Progress on an In-Flight Proxy Operation {#sec-PROXY_PROGRESS}

### ARGUMENTS

~~~ xdr
/// struct PROXY_PROGRESS4args {
///     uint64_t         ppa_registration_id;
///     uint64_t         ppa_operation_id;
///     bool             ppa_terminal;
///     nfsstat4         ppa_status;
///     uint64_t         ppa_bytes_done;
///     uint64_t         ppa_bytes_total;
/// };
~~~
{: #fig-PROXY_PROGRESS4args title="XDR for PROXY_PROGRESS4args"}

### RESULTS

~~~ xdr
/// struct PROXY_PROGRESS4res {
///     nfsstat4         ppr_status;
/// };
~~~
{: #fig-PROXY_PROGRESS4res title="XDR for PROXY_PROGRESS4res"}

### DESCRIPTION

A registered proxy server calls PROXY_PROGRESS on the
fore-channel of its session to the MDS to report the status
of an in-flight CB_PROXY_MOVE or CB_PROXY_REPAIR.  The PS MAY
send progress updates at any cadence it chooses.  The MDS uses
non-terminal updates for observability and liveness detection;
it MUST NOT require any specific update cadence as a
correctness condition.

The ppa_registration_id field identifies the PS's
registration.  The ppa_operation_id field identifies the
specific CB_PROXY_MOVE or CB_PROXY_REPAIR operation being
reported on; it is the value the PS returned in
cpmr_operation_id or cprr_operation_id when the MDS issued the
directive.

The ppa_terminal field distinguishes interim updates from the
final status for the operation:

-  When ppa_terminal is false, the update is an interim
   progress report.  The ppa_status field SHOULD be NFS4_OK
   (healthy progress) or NFS4ERR_DELAY (slower than expected
   but still progressing).

-  When ppa_terminal is true, the update is the final status
   for this operation.  The proxy MUST NOT subsequently issue
   PROXY_PROGRESS for the same ppa_operation_id.  Terminal
   values of ppa_status are:

   -  NFS4_OK: the destination layout is fully populated and
      consistent.  The MDS proceeds to CB_LAYOUTRECALL the old
      layout, waits for LAYOUTRETURNs, and retires the source
      DSes.

   -  NFS4ERR_PAYLOAD_LOST: for a CB_PROXY_REPAIR, the source
      layout was degraded beyond reconstructibility and the
      MDS MUST mark the affected byte ranges lost.  For a
      CB_PROXY_MOVE, a catastrophic loss of both source and
      destination during the move that cannot be recovered.

   -  Any other nfsstat4: proxy-specific failure.  The MDS MAY
      retry by reassigning the operation to a different
      registered PS (subject to the original deadline and any
      retry policy the MDS applies).

The ppa_bytes_done and ppa_bytes_total fields are advisory.
When the PS knows both (for example, moves on files with a
stable size), it SHOULD populate them.  When the total is not
known (for example, a source that is still being appended to),
the PS MUST set ppa_bytes_total to 0; ppa_bytes_done MAY still
carry cumulative progress.

The MDS returns ppr_status of NFS4_OK to acknowledge a
well-formed PROXY_PROGRESS, or an error if the registration or
operation id is unknown.  For terminal updates, the PS MUST
retry if it does not receive an acknowledgment within an
implementation-defined timeout; the MDS MUST treat duplicate
terminal updates for the same (ppa_registration_id,
ppa_operation_id) pair as idempotent.  Non-terminal updates
are fire-and-forget and need not be retried.

Cancellation of an in-flight operation by the MDS is handled
by CB_PROXY_CANCEL ({{sec-CB_PROXY_CANCEL}}).  Polled status
queries by the MDS use CB_PROXY_STATUS
({{sec-CB_PROXY_STATUS}}).

# New NFSv4.2 Callback Operations {#sec-new-cb-ops}

This document defines four new NFSv4.2 callback operations
that the MDS issues to a registered PS on the back-channel of
the PS -> MDS session defined in {{sec-design-session}}.
CB_PROXY_MOVE (17) and CB_PROXY_REPAIR (18) direct the PS to
move or reconstruct a file.  CB_PROXY_STATUS (19) lets the
MDS poll for status.  CB_PROXY_CANCEL (20) lets the MDS
cancel an in-flight operation.

~~~ xdr
/// /* New callback operations for the Data Mover (MDS -> PS) */
///
/// OP_CB_PROXY_MOVE        = 17;
/// OP_CB_PROXY_REPAIR      = 18;
/// OP_CB_PROXY_STATUS      = 19;
/// OP_CB_PROXY_CANCEL      = 20;
~~~
{: #fig-data-mover-cb-opnums title="Data Mover callback operation numbers"}

Opcodes 17 through 20 continue the callback-op range that
{{I-D.haynes-nfsv4-flexfiles-v2}} opens at 16 (CB_CHUNK_REPAIR).
If other in-flight NFSv4.2 extensions collide on these values
during IANA coordination, the final assignment will be
reconciled by the consuming RFC editor.

The following amendment blocks extend the nfs_cb_argop4 and
nfs_cb_resop4 dispatch unions from {{RFC7863}} with the new
callback ops.

~~~ xdr
/// /* nfs_cb_argop4 amendment block */
///
/// case OP_CB_PROXY_MOVE:
///     CB_PROXY_MOVE4args opcbproxymove;
/// case OP_CB_PROXY_REPAIR:
///     CB_PROXY_REPAIR4args opcbproxyrepair;
/// case OP_CB_PROXY_STATUS:
///     CB_PROXY_STATUS4args opcbproxystatus;
/// case OP_CB_PROXY_CANCEL:
///     CB_PROXY_CANCEL4args opcbproxycancel;
~~~
{: #fig-nfs_cb_argop4-amend title="nfs_cb_argop4 amendment block"}

~~~ xdr
/// /* nfs_cb_resop4 amendment block */
///
/// case OP_CB_PROXY_MOVE:
///     CB_PROXY_MOVE4res opcbproxymove;
/// case OP_CB_PROXY_REPAIR:
///     CB_PROXY_REPAIR4res opcbproxyrepair;
/// case OP_CB_PROXY_STATUS:
///     CB_PROXY_STATUS4res opcbproxystatus;
/// case OP_CB_PROXY_CANCEL:
///     CB_PROXY_CANCEL4res opcbproxycancel;
~~~
{: #fig-nfs_cb_resop4-amend title="nfs_cb_resop4 amendment block"}

## Callback Operation 17: CB_PROXY_MOVE - Direct a PS to Move a File {#sec-CB_PROXY_MOVE}

### ARGUMENTS

~~~ xdr
/// const CPM_FLAG_DUAL_WRITE = 0x00000001;
///
/// struct CB_PROXY_MOVE4args {
///     uint64_t         cpma_registration_id;
///     nfs_fh4          cpma_fh;
///     ffv2_layout4     cpma_source_layout;
///     ffv2_layout4     cpma_destination_layout;
///     nfstime4         cpma_deadline;
///     uint32_t         cpma_flags;
/// };
~~~
{: #fig-CB_PROXY_MOVE4args title="XDR for CB_PROXY_MOVE4args"}

### RESULTS

~~~ xdr
/// struct CB_PROXY_MOVE4resok {
///     uint64_t         cpmr_operation_id;
/// };
///
/// union CB_PROXY_MOVE4res switch (nfsstat4 cpmr_status) {
///     case NFS4_OK:
///         CB_PROXY_MOVE4resok cpmr_resok4;
///     default:
///         void;
/// };
~~~
{: #fig-CB_PROXY_MOVE4res title="XDR for CB_PROXY_MOVE4res"}

### DESCRIPTION

The MDS issues CB_PROXY_MOVE on the back-channel of a PS's
session to direct the PS to move a file from one layout to
another.  The move covers the whole file; partial-range moves
are out of scope for this revision.

The cpma_registration_id field MUST reference the registration
that the PS established on the same session via
PROXY_REGISTRATION.  An expired or mismatched registration
id MUST be rejected.

The cpma_fh field is the MDS filehandle of the file to move.

The cpma_source_layout and cpma_destination_layout fields are
full layout specifications.  The PS reads from the source
layout's DSes and writes to the destination layout's DSes.
Either layout MAY reference DSes not otherwise participating
in the operation; the PS acts as an NFSv4.2 client to each DS
named in either layout.  The transformation class (encode,
decode, transcode, or identity-with-relocation) is determined
entirely by the (source, destination) pair; no separate
transformation field is needed.

The cpma_deadline field is a wall-clock nfstime4 by which the
PS is expected to have completed the move.  Missing the
deadline does not corrupt state, but the MDS MAY cancel (via
CB_PROXY_CANCEL, {{sec-CB_PROXY_CANCEL}}) and reassign.

The cpma_flags field currently defines one flag,
CPM_FLAG_DUAL_WRITE.  When set, the move proceeds online: the
PS accepts client writes during the operation and replicates
each write to both the source and destination mirror sets,
and on completion the source is retired without rollback
risk.  When unset, the move is quiesced: the MDS recalls
client layouts for the duration, any client that opens the
file during the move is directed through the PS, and the PS
does not replicate to the source because no client is
writing.  The choice between the two modes is a
deployment-policy decision; PS implementations MUST support
the quiesced mode (the unset case) and SHOULD support
CPM_FLAG_DUAL_WRITE.

On success the PS returns a PS-assigned cpmr_operation_id for
the in-flight move, used for progress reporting
(PROXY_PROGRESS), status polling (CB_PROXY_STATUS), and
cancellation (CB_PROXY_CANCEL).

Terminal outcomes (communicated via ppa_status in a terminal
PROXY_PROGRESS, not as the return code of CB_PROXY_MOVE
itself):

-  NFS4_OK: destination fully populated and consistent.
-  NFS4ERR_PAYLOAD_LOST: source layout degraded beyond
   reconstructibility; operation aborted and the MDS marks
   affected byte ranges lost.
-  Other codes: PS-specific failure; the MDS MAY retry or
   reassign.

NFS4ERR_DELAY appears only in interim (non-terminal)
PROXY_PROGRESS updates to signal that the PS needs more time;
it MUST NOT be used as a terminal status.

## Callback Operation 18: CB_PROXY_REPAIR - Direct a PS to Reconstruct a File {#sec-CB_PROXY_REPAIR}

### ARGUMENTS

~~~ xdr
/// struct CB_PROXY_REPAIR4args {
///     uint64_t         cpra_registration_id;
///     nfs_fh4          cpra_fh;
///     ffv2_layout4     cpra_source_layout;
///     ffv2_layout4     cpra_destination_layout;
///     nfstime4         cpra_deadline;
/// };
~~~
{: #fig-CB_PROXY_REPAIR4args title="XDR for CB_PROXY_REPAIR4args"}

### RESULTS

~~~ xdr
/// struct CB_PROXY_REPAIR4resok {
///     uint64_t         cprr_operation_id;
/// };
///
/// union CB_PROXY_REPAIR4res switch (nfsstat4 cprr_status) {
///     case NFS4_OK:
///         CB_PROXY_REPAIR4resok cprr_resok4;
///     default:
///         void;
/// };
~~~
{: #fig-CB_PROXY_REPAIR4res title="XDR for CB_PROXY_REPAIR4res"}

### DESCRIPTION

CB_PROXY_REPAIR is a convenience specialization of
CB_PROXY_MOVE where the source layout is degraded.  The wire
shape is nearly identical; the separate op exists for
readability (MDS intent is apparent from the op name) and to
reserve the option of repair-specific progress semantics.

The argument fields are analogous to the corresponding fields
in CB_PROXY_MOVE4args, with cpra_source_layout understood to
describe a degraded layout -- one or more DSes in the source
layout are unreachable or hold shards that cannot be
validated.  The PS reconstructs the file's data from whatever
surviving shards the source layout references and writes the
reconstructed content to the destination layout.  If fewer
than k shards survive across the mirror set, the operation
terminates (via a terminal PROXY_PROGRESS) with
NFS4ERR_PAYLOAD_LOST for the affected byte ranges, matching
the per-chunk repair semantics in the Repair Client Selection
section of {{I-D.haynes-nfsv4-flexfiles-v2}}.

An implementation MAY collapse CB_PROXY_MOVE and
CB_PROXY_REPAIR into a single internal dispatch.
CB_PROXY_REPAIR is always quiesced (no CPM_FLAG_DUAL_WRITE
equivalent); client writes cannot be safely replicated until
the destination is consistent.

## Callback Operation 19: CB_PROXY_STATUS - Poll Status of an In-Flight Proxy Operation {#sec-CB_PROXY_STATUS}

### ARGUMENTS

~~~ xdr
/// struct CB_PROXY_STATUS4args {
///     uint64_t         cpsa_registration_id;
///     uint64_t         cpsa_operation_id;
/// };
~~~
{: #fig-CB_PROXY_STATUS4args title="XDR for CB_PROXY_STATUS4args"}

### RESULTS

~~~ xdr
/// struct CB_PROXY_STATUS4resok {
///     nfsstat4         cpsr_op_status;
///     uint64_t         cpsr_bytes_done;
///     uint64_t         cpsr_bytes_total;
/// };
///
/// union CB_PROXY_STATUS4res switch (nfsstat4 cpsr_status) {
///     case NFS4_OK:
///         CB_PROXY_STATUS4resok cpsr_resok4;
///     default:
///         void;
/// };
~~~
{: #fig-CB_PROXY_STATUS4res title="XDR for CB_PROXY_STATUS4res"}

### DESCRIPTION

The MDS MAY issue CB_PROXY_STATUS at any time to poll the
status of an in-flight CB_PROXY_MOVE or CB_PROXY_REPAIR.  The
op is the OFFLOAD_STATUS analog from {{RFC7862}}, adapted to
the MDS-as-poller direction.  It supplements PROXY_PROGRESS
(which is PS-initiated push) and is primarily useful when the
MDS has reason to believe it may have missed a progress
update or wants to check on a long-running operation without
waiting for the next push.

The cpsa_registration_id and cpsa_operation_id fields
identify the operation, using the registration id returned by
PROXY_REGISTRATION and the operation id returned by
CB_PROXY_MOVE or CB_PROXY_REPAIR.

On success, cpsr_op_status carries the current operational
status of the named operation.  Values mirror the ppa_status
space in PROXY_PROGRESS: NFS4_OK means healthy progress (or
completed successfully, if the PS happens to answer after
terminal state is reached), NFS4ERR_DELAY means the PS is
proceeding but slower than expected, NFS4ERR_PAYLOAD_LOST
means terminal failure, and other codes are PS-specific.

The cpsr_bytes_done and cpsr_bytes_total fields are advisory
and follow the same semantics as ppa_bytes_done and
ppa_bytes_total in PROXY_PROGRESS.

CB_PROXY_STATUS is a query; it does not change the state of
the named operation.  If the PS does not recognize the
(cpsa_registration_id, cpsa_operation_id) pair (because the
operation has completed and been garbage-collected, or never
existed), the PS MUST return the top-level cpsr_status as
NFS4ERR_NOENT.

## Callback Operation 20: CB_PROXY_CANCEL - Cancel an In-Flight Proxy Operation {#sec-CB_PROXY_CANCEL}

### ARGUMENTS

~~~ xdr
/// struct CB_PROXY_CANCEL4args {
///     uint64_t         cpca_registration_id;
///     uint64_t         cpca_operation_id;
/// };
~~~
{: #fig-CB_PROXY_CANCEL4args title="XDR for CB_PROXY_CANCEL4args"}

### RESULTS

~~~ xdr
/// struct CB_PROXY_CANCEL4res {
///     nfsstat4         cpcr_status;
/// };
~~~
{: #fig-CB_PROXY_CANCEL4res title="XDR for CB_PROXY_CANCEL4res"}

### DESCRIPTION

The MDS issues CB_PROXY_CANCEL to direct a PS to abort an
in-flight CB_PROXY_MOVE or CB_PROXY_REPAIR.  The op is the
OFFLOAD_CANCEL analog from {{RFC7862}}, adapted to the
MDS-as-canceller direction.

On receipt of CB_PROXY_CANCEL for a known operation, the PS
MUST stop the operation as soon as practical and MUST issue a
terminal PROXY_PROGRESS with an implementation-chosen
nfsstat4 other than NFS4_OK (for example, NFS4ERR_DELAY is
inappropriate; a PS-specific "cancelled by MDS" code or
NFS4ERR_SERVERFAULT is appropriate).  The PS MAY leave the
destination in a partial state; the MDS is responsible for
any cleanup of destination DSes the PS populated before
cancellation, either by issuing a follow-on CB_PROXY_MOVE to
a different destination, by reverting to the pre-move source
layout, or by marking destination DSes for cleanup.

On receipt of CB_PROXY_CANCEL for an unknown operation
(completed and garbage-collected, or never issued), the PS
MUST return cpcr_status NFS4ERR_NOENT.

Cancellation is a best-effort directive.  A PS that has
already reached a terminal state internally (but has not yet
delivered its terminal PROXY_PROGRESS) MAY acknowledge
CB_PROXY_CANCEL and then deliver the original terminal status
anyway; the MDS MUST treat this as equivalent to the original
terminal outcome.

# Affinity Matching {#sec-affinity}

A PS MAY populate prr_affinity at registration time with a
token that lets the MDS recognize co-residency with a client.
The canonical use case: a client host is running a local PS,
potentially in a container on the same physical machine as
the client.  Network-level checks alone are not reliable --
the PS may be on a container IP, a separate netns, or behind
a NAT -- so an explicit token is required.

The matching algorithm has three moving parts.  First,
clients opt in by embedding an affinity token in the
co_ownerid they present at EXCHANGE_ID; clients that do not
wish to participate send a normal co_ownerid with no embedded
token.  Second, when the MDS processes a LAYOUTGET on a file
that has an active CB_PROXY_MOVE or CB_PROXY_REPAIR, the MDS
iterates over registered PSes and compares each PS's
prr_affinity against the requesting client's co_ownerid; a
match (equality, substring, or hash equivalence -- the match
predicate is implementation-defined but MUST be deterministic
for a given MDS instance) indicates co-residency.  Third,
when a match exists the MDS MAY use it as input to PS
selection, preferring, when multiple PSes are eligible, the
one that matches the most clients' affinity tokens.

The layout returned names the selected PS first in
ffs_data_servers.  For mirrored coding types this makes the
local PS the client's default read target (matching the FFv1
affinity pattern).  For erasure-coded types it makes the
local PS the preferred encode endpoint.

Affinity is advisory.  The MDS MUST NOT grant any authority
based solely on affinity; the normal authentication model
still applies, and a client that claims an affinity token it
has no right to gains at most a sub-optimal layout, not
unauthorized access.

Note: there is no proposal to stamp the affinity token into the
filehandle.  The MDS has already performed the match before the
layout reaches the client, and the layout's deviceinfo carries
all the information a client-side shortcut needs.

# Layout Shape During a Proxy Operation {#sec-layout-shape}

The layout the MDS hands out to clients while a proxy
operation is active is the mechanism's sole client-facing
surface.  Everything else in this document -- the session,
the ops, the credential-forwarding rules -- is between the
MDS and the PS.  The layout shape is therefore what a client
implementer needs to read to know how its code interacts with
a proxied file.

When a CB_PROXY_MOVE or CB_PROXY_REPAIR is active for a file,
the layout the MDS hands out to clients contains a **proxy
DS entry** at the head of ffs_data_servers (or otherwise
flagged for routing; see the flag below).  This entry names
the selected PS.  Source and destination DS entries MAY also
appear in the layout at the MDS's discretion, but in the
simplest case the PS is the only visible DS, and the source
and destination mirror sets are internal to the PS.

The PROXY flag is a new bit on ffv2_ds_flags4:

~~~ xdr
/// const FFV2_DS_FLAGS_PROXY         = 0x00000040;
~~~
{: #fig-FFV2_DS_FLAGS_PROXY title="FFV2_DS_FLAGS_PROXY" }

As per {{I-D.haynes-nfsv4-flexfiles-v2}}'s Flag-Word Allocation
rule, IANA does not maintain a registry for ffv2_ds_flags4; the
bit is governed by this document.

When FFV2_DS_FLAGS_PROXY is set on any data server entry in
a layout, clients MUST direct all CHUNK I/O for this file to
that DS rather than to any other data server in the layout.
The PS internally dispatches reads and writes to the source
and destination DSes.

## Single-Layout Model

This design uses a single layout with a PROXY-flagged entry
rather than two linked layouts.  Three considerations drive
that choice.  pNFS clients already handle single layouts
cleanly, so no new layout-linkage mechanism needs to be
invented or implemented on the client.  The client's view of
the file -- "the file's DS is the PS" -- is the truth during
the operation, so exposing the source and destination DSes
directly would invite confusion about which entry to address
rather than clarify.  And late-arriving clients see the proxy
layout from the start, without any separate setup path to
join an operation already in progress.  The alternative -- a
source layout plus a destination layout linked by a
redirector record -- was considered and rejected on those
three grounds.

# Client Behavior

A client that observes a layout with FFV2_DS_FLAGS_PROXY
routes all CHUNK I/O to the PROXY-flagged DS entry and does
not issue I/O directly to any non-PROXY DS in that layout.
Non-PROXY DSes MAY appear in the layout for informational
reasons but MUST NOT be addressed by the client.

The client uses its existing layout stateid against the
PROXY-flagged DS entry; the PS accepts CHUNK ops under that
stateid because the MDS has registered the stateid via
TRUST_STATEID on the PS, per the tight-coupling semantics in
{{I-D.haynes-nfsv4-flexfiles-v2}}.

The client handles PS-side errors (NFS4ERR_DELAY, connection
loss, NFS4ERR_BAD_STATEID) exactly as it would any other DS
error: report LAYOUTERROR to the MDS and expect either a new
layout or a PS reassignment in return.

## When the Layout Is Recalled

If the MDS recalls the layout mid-operation (the PS failed
and is being replaced, or the operation completed and normal
DS layouts are being reissued), the client LAYOUTRETURNs as
usual and reacquires via LAYOUTGET.  The new layout may name
a different PS, a different mirror set, or no PROXY-flagged
entry if the operation has completed.

## In-Flight I/O When the PS Changes

In-flight I/O to the old PS when the MDS recalls the layout
MAY complete at the old PS; results remain valid under the
old PS's authority.  New I/O issued after LAYOUTRETURN MUST
go through the replacement PS (or, if the new layout has no
PROXY-flagged entry, directly to the DSes named there).

# State Machine

A file's participation in a proxy operation passes through
four states: READY (no operation in flight), PROXY_ACTIVE
(the PS is driving a move or repair), COMMITTING (the PS has
reported terminal success and the MDS is recalling the old
layout), and DONE (clients are on the post-move layout,
source DSes retired).  The state is MDS-local: clients never
observe these state names directly, but a client's behaviour
is shaped by which layout the MDS is currently handing out.
A given file spends most of its lifetime in READY; a proxy
operation is a relatively short excursion through the other
three states, after which the file returns to READY with a
new layout in place (or, on cancellation or failure, with
the old layout preserved).

The diagram below shows the happy-path progression; the
table that follows enumerates every state transition
including the unhappy ones (cancellation, PS failure without
replacement).

~~~
            (admin, policy, repair, or maintenance trigger)
                               |
                               v
                         +------------+
                         |   READY    |
                         | source     |
                         | layout     |
                         | only       |
                         +-----+------+
                               |
                               | MDS selects registered PS,
                               | issues CB_PROXY_MOVE (or
                               | CB_PROXY_REPAIR) on the
                               | PS session's back-channel
                               v
                         +--------------+
                         | PROXY_ACTIVE |
                         | clients see  |
                         | layout with  |
                         | PS DS at     |
                         | head of      |
                         | ffs_data_    |
                         | servers;     |
                         | PS drives    |
                         | source->dest |
                         +-----+--------+
                               |
                               | PS issues terminal
                               | PROXY_PROGRESS
                               | (ppa_terminal=true,
                               |  ppa_status=NFS4_OK)
                               v
                         +------------+
                         | COMMITTING |
                         | MDS issues |
                         | CB_LAYOUT- |
                         | RECALL for |
                         | old layout |
                         +-----+------+
                               |
                               | all clients have
                               | LAYOUTRETURNed
                               v
                         +------------+
                         |   DONE     |
                         | new layout |
                         | live;      |
                         | source     |
                         | DSes       |
                         | retired    |
                         +------------+
~~~

## Transitions

| From | To | Trigger | Actions |
|------|-----|---------|---------|
| READY | PROXY_ACTIVE | MDS decides to move or repair | MDS issues CB_PROXY_MOVE or CB_PROXY_REPAIR on the PS session's back-channel; MDS starts handing out layouts with FFV2_DS_FLAGS_PROXY set on the PS entry |
| PROXY_ACTIVE | COMMITTING | PS issues terminal PROXY_PROGRESS with ppa_status=NFS4_OK | MDS begins CB_LAYOUTRECALL fan-out to clients still on the old layout |
| COMMITTING | DONE | All clients have LAYOUTRETURNed | MDS issues post-move layouts; source DSes retired |
| PROXY_ACTIVE | READY | PS failed and no replacement available; or MDS-initiated cancellation via CB_PROXY_CANCEL | MDS reverts layouts to pre-move source set |

# PS Failure and Recovery

## PS Crash During PROXY_ACTIVE

When a PS crashes mid-operation, client I/O routed through
it receives NFS4ERR_DELAY (if the PS is reachable but
unhealthy) or connection errors (if unreachable), and the
affected clients report LAYOUTERROR to the MDS.  The MDS MAY
select a replacement PS from the registered pool and issue
CB_PROXY_MOVE or CB_PROXY_REPAIR to the replacement, with the
source layout updated to reflect current reality --
destination DSes that the failed PS populated are now part of
the source set -- and the destination layout unchanged; the
replacement PS resumes from wherever the failed PS left off.
The MDS then issues CB_LAYOUTRECALL on the old layout and the
replacement PS's layout becomes live for new LAYOUTGETs.

If the MDS cannot find a replacement within a policy timeout,
it MUST cancel the operation: revert to the pre-move source
layout, do not issue a destination layout, and mark the
destination DSes for cleanup or retry.

## Cascading PS Failure

Repeated PS failures on the same operation SHOULD trigger
escalation to deployment management rather than recursive
retry.  Recurring failures likely indicate an environmental
issue the PS cannot work around.

## Source DS Crash During PROXY_ACTIVE

Reduces the PS's read parallelism but does not block forward
progress as long as the erasure code can still reconstruct
from surviving source DSes.  If the source degrades past
reconstructibility, the operation transitions to whole-file
repair semantics automatically: partial reconstruction
succeeds; ranges that cannot be reconstructed terminate the
operation with NFS4ERR_PAYLOAD_LOST.

## Destination DS Crash During PROXY_ACTIVE

Treated as a normal DS failure on the destination side.  The
PS acts like a client to the destination DSes: LAYOUTERROR to
the MDS, which MAY substitute a spare or mark the destination
FFV2_DS_FLAGS_REPAIR.  The PS continues pushing to the
remaining destinations.  Clients are unaffected.

# MDS Crash Recovery

Clients and the PS detect MDS session loss and enter RECLAIM
per {{RFC8881}}.  The PS reclaims its PROXY_REGISTRATION on
reconnection; the MDS MAY persist registrations across
restart, or the PS MAY re-register from scratch.
PROXY_REGISTRATION is idempotent as long as the PS preserves
its prr_registration_id.  The MDS re-advertises proxy layouts
(with FFV2_DS_FLAGS_PROXY) to reclaiming clients.

If the MDS persisted its view of the active CB_PROXY_MOVE and
CB_PROXY_REPAIR operations -- which destination DSes are
populated, which chunks are committed, and similar detail --
the PS resumes from the last persisted checkpoint.
Persistence is RECOMMENDED but not required; a non-persistent
MDS restarts the operation from the beginning.

A subtle window exists between the PS issuing a terminal
PROXY_PROGRESS and the MDS issuing CB_LAYOUTRECALL.  If the
MDS crashed in that window, the MDS on restart SHOULD
re-drive the COMMITTING phase.  The PS MAY re-issue its
terminal PROXY_PROGRESS on session re-establishment; the MDS
MUST treat a re-issued terminal update as idempotent per the
retry rules in {{sec-PROXY_PROGRESS}}.

# Backward Compatibility

## Clients

Client behavior is a normal layout path with a new flag.
Clients that do not recognize FFV2_DS_FLAGS_PROXY will treat
the PROXY-flagged DS entry as any other DS and route I/O to
it normally; that is in fact the correct behavior.  No
client-side version negotiation is needed.

Clients that require strict per-DS identity checking (e.g.,
"the layout's DS must match a pre-allowlisted fingerprint")
should extend their allowlist to include registered proxy
servers.  This is a deployment concern, not a protocol one.

## Data Servers

A host that does not implement the proxy server role simply
does not call PROXY_REGISTRATION and is never selected for a
CB_PROXY_MOVE or CB_PROXY_REPAIR.  A deployment with no
registered PS falls back to per-chunk CB_CHUNK_REPAIR for
single-shard repair, to admin-coordinated offline procedures
for policy transitions and DS evacuation, and to blocking
DS maintenance -- the DS cannot drain through a PS, so it
must remain reachable to clients throughout its service
life.

Deployments SHOULD ensure at least one registered PS exists
per failure domain to avoid a single point of failure on
move operations.

## NFSv3 Source DSes

When the source mirror is an NFSv3 DS, the PS reads from it
using NFSv3 semantics and writes to the NFSv4.2 destination
using CHUNK semantics.  This is the same pattern
{{I-D.haynes-nfsv4-flexfiles-v2}} uses for InBand I/O.

# Security Considerations {#sec-security}

The security surface added by this document sits in two
places: the session the PS establishes with the MDS, and the
data path clients take through the PS during a proxy
operation.  The session is narrower than the data path --
only the MDS talks to the PS over it, and the MDS has long
been a trusted coordinator in the pNFS model -- but it
carries operations that affect every client whose layouts
reach a PS.  The data path is broader, because it exposes
the PS to every client whose layout has the FFV2_DS_FLAGS_PROXY
flag; a compromised PS on that path has the same observational
and modification reach as a compromised DS, and in the
translating-PS case a larger reach because of the elevated
identity the PS typically runs with.

The numbered items below name the specific threats the
design either addresses or explicitly leaves out of scope.
The rule on credential forwarding, because it is the most
consequential and the most easily implemented incorrectly,
is expanded in {{sec-credential-forwarding}}.

1.  **PS authority.**  A PS in PROXY_ACTIVE sees all client
    I/O for the proxied file.  A compromised PS can observe
    or modify file data.  Deployments MUST treat PS-capable
    hosts as at least as trusted as the DSes they proxy for.
    PROXY_REGISTRATION SHOULD be gated by a deployment-level
    allowlist; arbitrary hosts that present the op without
    prior provisioning SHOULD be rejected.

2.  **Transport security across the operation.**  The PS's
    connections to source and destination DSes are
    independent of the client's connection to the PS.  A PS
    MAY read from an AUTH_SYS source and write to a TLS
    destination (or any other combination).  The PS is
    responsible for enforcing the effective security policy
    (e.g., do not downgrade encrypted data to a plaintext
    DS).

3.  **Principal binding during a proxy operation.**  For
    PS-to-DS traffic (the PS reading source DSes and writing
    destination DSes to carry out a CB_PROXY_MOVE or
    CB_PROXY_REPAIR), the PS presents a principal to those
    DSes that they will accept; this is the PS's own service
    identity unless constrained delegation or equivalent is
    arranged.  Forwarding the client's identity to the peer
    DSes for PS-driven data movement is NOT required and is
    typically NOT practical (the client is not in the
    conversation at that point).  See the Credential
    Forwarding and Privilege Boundary section below for the
    case of client-initiated file I/O through a translating
    PS, where the credential-forwarding rule is different and
    stricter.

4.  **PS impersonation.**  A malicious MDS could register a
    hostile entity as a PS.  The existing MDS trust model
    already grants the MDS this capability via
    CB_LAYOUTRECALL and the ability to issue any layout it
    chooses; PROXY_REGISTRATION does not weaken it.  Clients
    that require stronger PS identity verification SHOULD
    validate the PS's transport-security credentials against
    a deployment allowlist.

5.  **Affinity token.**  prr_affinity is a co-residency
    attestation, not an authentication mechanism.  Matching
    an affinity token between a PS and a client grants the
    client no new access rights; it is used only as input to
    PS-selection preferences.  A client cannot elevate
    privilege by spoofing an affinity token.

6.  **Registration lease expiry.**  If a PS's lease expires
    mid-operation, the MDS MUST cancel the operation (via
    CB_PROXY_CANCEL, followed by layout revert and cleanup of
    destination DSes).  The MDS MUST NOT continue to route
    client I/O to a PS whose registration has lapsed.

## Credential Forwarding and the Privilege Boundary {#sec-credential-forwarding}

A translating PS (see {{sec-codec-translation}}) has
structurally elevated privilege by design.  To perform its
management tasks -- moves, repairs, evacuations,
cross-tenant re-exports -- the deployment grants the PS's
service identity broad access: typically not-root-squashed,
often read/write to every file in the namespace, and session
authority to every DS.  That privilege is intentional.

A codec-ignorant client that reaches the PS, however, arrives
with its own RPC credentials that the PS does not itself need
in order to function.  An NFSv3 client's uid/gid, an
AUTH_SYS-squashed identity, an RPCSEC_GSS principal -- none of
these are the PS's own.  If the PS ignores the client's
credentials and issues MDS or DS operations under its own
service identity when translating client I/O, every client
that reaches the PS silently inherits the PS's privilege.
This is a protocol-level privilege-escalation vector, and
this document calls it out rather than hiding it.

The normative requirements below apply whenever a PS is
translating client-initiated file I/O (as distinct from
PS-driven move / repair work, which runs under the PS's own
authority on directives from the MDS).  They form a cohesive
set: credential pass-through (rule 1) is the core
requirement; no-squash-inversion (rule 2) closes the most
common way rule 1 can be implemented incorrectly;
authorization-remains-with-MDS (rule 3) names the
responsibility on the MDS side of the same contract;
service-identity-is-for-the-control-plane (rule 4) draws the
line between the op paths where the PS uses its own
credentials and the op paths where it does not; and the
failure-mode rule (rule 5) specifies the correct refusal
behaviour rather than letting a silent fall-through become
the escape hatch.

1.  **Credential pass-through.**  The PS MUST present the
    client's credentials (RPC auth flavor and principal) on
    every MDS or DS operation it issues as a consequence of a
    client-initiated request.  Specifically, a client `READ`
    that the PS expands into `LAYOUTGET` + `CHUNK_READ` MUST
    carry the client's credentials on both the `LAYOUTGET`
    against the MDS and the `CHUNK_READ` against the DSes.
    The PS MUST NOT substitute its own service identity for
    client-initiated operations.

2.  **No squash inversion.**  If the client arrives with a
    root-squashed identity (for example, uid 0 mapped to
    nobody by the NFSv3 export configuration on the
    client-facing side of the PS), the PS MUST preserve the
    squashed identity when forwarding.  The PS MUST NOT
    translate a client's squashed credentials back into
    unsquashed root, even though the PS's own identity is
    typically unsquashed.

3.  **Authorization remains with the MDS.**  When a client-
    initiated operation reaches the MDS over a PS <-> MDS
    session, the MDS MUST use the RPC credentials carried on
    that compound for authorization and MUST NOT substitute
    the PS's session-level identity.  Equivalently: the MDS
    performs access-control checks against the forwarded
    client credentials, not against the PS's service
    identity, for any client-initiated file operation.  The
    PS is a translator, not an authority.  This is what
    prevents PS deployment from becoming a blanket ACL
    override.

4.  **PS service identity is for the control plane only.**
    The PS MAY, and typically MUST, use its own service
    identity for:
    -  The MDS <-> PS session (the session the PS opens to
       the MDS, on which PROXY_REGISTRATION and PROXY_PROGRESS
       flow on the fore-channel and CB_PROXY_MOVE /
       CB_PROXY_REPAIR / CB_PROXY_STATUS / CB_PROXY_CANCEL
       flow on the back-channel).
    -  Peer-DS session setup for PS-driven data movement
       (reading source DSes, writing destination DSes under
       a CB_PROXY_MOVE that the MDS has authorized).
    -  PS housekeeping.

    The PS's service identity MUST NOT be used for
    client-initiated file data operations.

5.  **Failure mode on missing credentials.**  If the PS
    cannot forward a client's credentials for some reason
    (e.g., the client presented AUTH_NONE, or the
    client-facing side used a security flavor the PS cannot
    propagate), the PS MUST reject the client operation with
    the equivalent of NFS4ERR_ACCESS (or NFS3ERR_ACCES for
    NFSv3 clients).  The PS MUST NOT fall back to serving the
    operation under its own identity.

Deployment-level requirements:

-  PROXY_REGISTRATION MUST be allowlisted.  An unknown host
   presenting PROXY_REGISTRATION MUST be rejected.  This is
   the only wire-level defense against a hostile entity
   registering as a PS and then receiving client-forwarded
   credentials.

-  The MDS <-> PS session MUST use RPCSEC_GSS {{RFC7861}} or
   RPC-over-TLS {{RFC9289}} with mutual authentication.
   AUTH_SYS on the MDS <-> PS session is forbidden.

-  Deployments SHOULD audit both the PS's credential-
   forwarding behavior (the PS logs what it forwards) and
   the MDS's authorization checks (the MDS logs what
   principal authorized each operation).  Divergence between
   the two indicates a credential-forwarding bug or
   compromise.

What the protocol cannot defend against:

-  A compromised PS has direct access to whatever credentials
   pass through it.  Credential confidentiality collapses the
   moment the PS is under adversary control.  Mitigation is
   operational: restrict which hosts can register as a PS,
   audit PROXY_REGISTRATION events, rotate deployment-level
   keys.

-  A deployment that configures a PS to run as root while
   the client is root-squashed has already violated rule 2
   above; no wire mechanism detects a PS deliberately
   mis-implementing credential forwarding.  Deployments
   SHOULD verify their PS implementation's credential-
   forwarding behavior through conformance testing before
   production use.

Future work (noted as an Open Question below): RPCSEC_GSSv3
structured privilege assertion per {{RFC7861}} Section 2.5.2
is the natural strong-authentication mechanism for
PS-forwarded credentials.  This revision does not require
GSSv3 because the broader NFSv4 deployment base does not yet
support it; deployments that can use GSSv3 SHOULD prefer it
over AUTH_SYS passthrough for the credential-forwarding
channel.

## Namespace Traversal Privilege {#sec-namespace-traversal-privilege}

A PS that translates client I/O has to know how the MDS's
namespace is shaped: which paths are exported, what filehandle
each path resolves to, how the exports mount within one another.
The PS acquires this information by traversing the MDS's
namespace -- LOOKUP, LOOKUPP, PUTFH, PUTROOTFH, GETFH on the
PS <-> MDS session.

This traversal cannot always run under forwarded client
credentials: at the point the PS needs to discover a new export
(a client has not yet asked for it, or the PS has just restarted
and has no FH cache) there is no client whose credentials the PS
could forward.  Deployments have two choices for how the PS
acquires namespace shape:

1.  **Grant a narrow traversal privilege.**  The MDS MAY treat
    a registered PS's service identity as authorized for
    LOOKUP, LOOKUPP, PUTFH, PUTROOTFH, GETFH, and SEQUENCE on
    the PS <-> MDS session without applying the MDS's
    export-rule filtering that would normally gate those
    names.  This is strictly a structural privilege: it
    permits the PS to see that paths exist and to obtain their
    filehandles, but grants no data access.  All operations
    that carry or require data authorization (OPEN, READ,
    WRITE, LAYOUTGET, GETATTR of privileged attributes, etc.)
    MUST still run under the rules of
    {{sec-credential-forwarding}}: forwarded client
    credentials for client-initiated operations, and PS
    service identity only for control-plane operations.

    A deployment that grants this privilege discloses the
    MDS's namespace shape to the PS's service identity --
    specifically, names that the PS's source address would
    not be able to see through the MDS's normal export
    filtering.  Deployments SHOULD audit traversal compounds
    on registered-PS sessions so the disclosure is reviewable;
    the MDS SHOULD log each LOOKUP / GETFH that benefits from
    the bypass.

2.  **Do not grant the privilege.**  The PS is required to
    translate every client-originated LOOKUP into a separate
    LOOKUP against the MDS under the forwarding client's
    credentials, caching only what the client's credentials
    authorized the MDS to return.  This eliminates the
    namespace-shape disclosure but costs an extra MDS
    round-trip per client LOOKUP-miss and leaves the PS
    unable to pre-discover exports.

This document does not normatively prefer one approach over
the other.  Implementations SHOULD document which they use;
deployment guidance for the common combined DS+PS case is
that option (1) is expected and the narrower privilege is
acceptable given the PS is already a trusted control-plane
peer of the MDS.

The traversal privilege is distinct from and narrower than
root_squash bypass.  A forwarded-uid-0 client operation (OPEN,
READ, etc.) under option (1) is still subject to normal
root_squash handling on the PS's source-address rule at the
MDS; the traversal privilege applies only to the six ops
enumerated above.

## PS-Side Policy Enforcement (informative) {#sec-ps-side-policy-enforcement}

A PS implementation MAY perform per-client export-rule
enforcement locally, rejecting operations the MDS would also
reject before forwarding them.  This is a performance
optimization: it keeps bad requests off the PS <-> MDS wire and
lets the PS return NFS4ERR_WRONGSEC / NFS4ERR_ACCESS without
paying a round-trip.

Local enforcement is not a security boundary.  Rule 3 of
{{sec-credential-forwarding}} names the MDS as the authority
for every client-initiated file operation.  A PS that performs
local enforcement is checking its own cached copy of the MDS's
per-client rules; if the copy is stale, wrong, or absent, the
PS MUST forward the operation and let the MDS decide.  A PS
implementation that declines to perform local enforcement is
conformant with this specification.

Deployments that want local enforcement need a mechanism for
the PS to acquire the MDS's per-export client-rule list.  This
document does not standardise such a mechanism; implementation-
specific options include reffs-style probe-protocol extensions,
out-of-band admin distribution, or a future revision of this
specification.  Any such mechanism MUST itself be authenticated
against the PS allowlist (the rules are sensitive deployment
policy) and MUST support a refresh path so PSes see rule changes
within a bounded time.

# IANA Considerations {#iana-considerations}

This document does not require any IANA action.

The two new NFSv4.2 operations defined in {{sec-new-ops}}
(OP_PROXY_REGISTRATION = 93, OP_PROXY_PROGRESS = 94) and the
four new NFSv4.2 callback operations defined in
{{sec-new-cb-ops}} (OP_CB_PROXY_MOVE = 17,
OP_CB_PROXY_REPAIR = 18, OP_CB_PROXY_STATUS = 19,
OP_CB_PROXY_CANCEL = 20) follow the convention that NFSv4.2
operation numbers are governed by the publishing document and
do not require a separate IANA registry entry.  The same
convention applies to the new flag bit FFV2_DS_FLAGS_PROXY,
which is an additional bit in the ffv2_ds_flags4 bitmap
defined by {{I-D.haynes-nfsv4-flexfiles-v2}}; that document
explicitly records its flag-word bitmaps as not
IANA-registered, and any future bit allocations are made by a
document that updates or obsoletes it.

The CPM_FLAG_DUAL_WRITE bit in cpma_flags (defined in
{{sec-CB_PROXY_MOVE}}) is a bit in a bitmap this document
introduces.  Following the precedent in
{{I-D.haynes-nfsv4-flexfiles-v2}} (which in turn follows
{{RFC8435}}), this document does not establish an IANA
registry for its bit spaces; future bit allocations are made
by a document that updates or obsoletes this one.
Implementations MUST treat unknown bits as reserved and MUST
NOT assign meaning to them locally.

# Interaction with the Main Draft {#interaction}

The mechanism this document specifies is built on top of
four constructs that {{I-D.haynes-nfsv4-flexfiles-v2}}
defines: the chunk_guard4 compare-and-swap primitive, the
CHUNK_LOCK mechanism, the CB_CHUNK_REPAIR per-chunk repair
callback, and the TRUST_STATEID / REVOKE_STATEID control
plane.  None of these are modified or extended here; this
section states how each is used (or explicitly excluded)
when a PS is active on a file.  Two of the four
(chunk_guard4, CHUNK_LOCK) describe what the PS does on the
DS side of the mechanism; the other two (CB_CHUNK_REPAIR,
TRUST_STATEID) describe how MDS-side bookkeeping composes
with a live proxy operation.

## chunk_guard4

The PS enforces chunk_guard4 CAS on the destination mirror
set on behalf of clients.  The PS MAY use the same guard
values client writes carry through it, or generate fresh
guard values on the destination side, provided uniqueness on
the destination is preserved.

## CHUNK_LOCK

If a client holds a chunk lock on a file when a proxy
operation activates, the lock follows the file: the PS takes
ownership of the lock on the destination mirror set, and the
MDS-escrow semantics (the Reserved cg_client_id Value
subsection of {{I-D.haynes-nfsv4-flexfiles-v2}}) apply if the
original holder becomes unreachable during the operation.

## CB_CHUNK_REPAIR

Per-chunk CB_CHUNK_REPAIR and a CB_PROXY_MOVE or
CB_PROXY_REPAIR on the same file are mutually exclusive at
any given time.  The MDS MUST NOT issue CB_CHUNK_REPAIR for a
file currently in PROXY_ACTIVE; the PS handles any mid-move
repair internally.  If the MDS decides a proxied file also
needs per-chunk repair after the proxy operation completes,
it issues CB_CHUNK_REPAIR against the post-move layout.

## TRUST_STATEID / REVOKE_STATEID

When the MDS selects a PS for a proxy operation, it issues
TRUST_STATEID on the PS for every client layout stateid that
will route through the PS during PROXY_ACTIVE.  On PS
retirement the MDS issues REVOKE_STATEID on the retired PS.
This is the same mechanism {{I-D.haynes-nfsv4-flexfiles-v2}}
defines for any DS in a tightly coupled deployment.

# Open Questions {#sec-open-questions}

The design is substantially complete but still has open
points that need Working Group input or internal agreement
before the first submission.  They fall into three rough
categories: wire-level details that need to be nailed down
(renewal semantics, the affinity match predicate, richer
capability advertising), architectural choices that affect
the mechanism's shape (multiple concurrent proxies per file,
transitive proxy, capability-scoped EXCHGID flag), and
policy questions whose answers bind deployment choices more
than wire behaviour (MDS operation-state persistence,
RPCSEC_GSSv3 requirement level, DEVICEID_REGISTRATION
generalization).  Each item below briefly states the
question and the candidate resolutions; none of them block
this document's core mechanism but each may reshape a detail
of it.

1.  **Registration renewal semantics.**  Is renewal a fresh
    PROXY_REGISTRATION with the same prr_registration_id
    (idempotent), or a separate PROXY_RENEW op (lighter-
    weight)?  The idempotent-reregistration path keeps the op
    count smaller and keeps renewal and first-time
    registration on a single code path, at the cost of
    carrying codec and affinity fields on every renewal.  A
    dedicated PROXY_RENEW op is cheaper on the wire but adds
    an op number and a second code path.  The choice mostly
    affects implementation complexity, not protocol
    expressiveness.

2.  **Affinity match predicate.**  Is exact equality of
    prr_affinity and co_ownerid sufficient, or does the spec
    need to define substring / hash matching explicitly?  If
    implementation-defined, MDS implementations that pick
    different predicates will produce different layouts --
    is that acceptable?

3.  **Multiple concurrent proxies per file.**  The design
    assumes one proxy per file per operation.  Should two
    proxies be allowed to pipeline a large file (proxy A
    drives the first 1 TB, proxy B drives the next)?  The
    motivating case is a multi-terabyte move where a single
    proxy's bandwidth is the bottleneck; parallelizing across
    proxies would shorten the operation proportionally.  The
    cost is state-machine complexity (two operation ids to
    track, partial-completion bookkeeping, range ownership
    between proxies) and layout complexity (the client sees
    two PS entries in ffs_data_servers and needs routing
    rules between them).  One-proxy-per-file keeps the
    mechanism simple; if the bandwidth case turns out to
    dominate in practice, a follow-on extension can add
    parallelism later without invalidating the single-proxy
    path.

4.  **Transitive proxy.**  If a file in PROXY_ACTIVE needs a
    second move (e.g., a DS maintenance window opens while a
    repair is already running), what happens?  Queueing the
    second move postpones the maintenance, which may not be
    acceptable if the maintenance window is hard.  Aborting
    the first move wastes the repair work already done and
    puts the file back into a degraded state.  Allowing a
    proxy to act as the source for another proxy (a "chained"
    proxy setup) preserves the repair progress but doubles the
    state-machine work and introduces failure-mode compounds
    that the current design does not cover.  The right answer
    probably depends on operator priorities and may need to be
    a configurable MDS policy rather than a protocol rule.

5.  **Persistent vs ephemeral MDS operation state.**  Is
    operation persistence a SHOULD or a MAY?  Production
    deployments probably want SHOULD to avoid restart cost on
    large moves; prototypes probably want MAY.

6.  **Registration as a capability-scoped authority.**  Should
    PROXY_REGISTRATION require a separate EXCHGID4 flag (e.g.,
    EXCHGID4_FLAG_USE_PROXY_DS) to distinguish proxy-capable
    DSes from generic DSes, or is the registration itself the
    capability declaration?

7.  **Richer capability advertising.**  prr_codecs covers the
    transformation classes that matter for move / repair.
    Features that are implementation-internal (encryption,
    compression, alignment normalization) do not need to be
    advertised because they do not affect the wire contract.
    Features that DO affect the wire (e.g., support for some
    future sparse-read or TRIM op) would warrant a richer
    capability descriptor.  Worth revisiting when those ops
    are defined.

8.  **RPCSEC_GSSv3 for translating-proxy credential
    forwarding.**  Credential forwarding under AUTH_SYS is
    weak (uid spoofable, no integrity protection).  RPCSEC_GSSv3
    structured privilege assertion ({{RFC7861}} Section 2.5.2)
    is the natural strong-authentication mechanism, but its
    deployment base in the NFSv4 community is narrow.  Should
    the draft REQUIRE GSSv3 for translating proxies, RECOMMEND
    it, or leave it as implementation-optional?  The answer
    likely depends on how aggressively the WG wants to push
    GSSv3 adoption as a side effect of standardizing this
    mechanism.

9.  **DEVICEID_REGISTRATION generalization.**
    PROXY_REGISTRATION in this document is a proxy-specific
    capability-advertisement op: a DS opens a session to the
    MDS and declares that it is proxy-capable, along with
    codec-set membership, an affinity token, and a lease.

    The same mechanism has broader applicability as a generic
    DS -> MDS capability advertisement -- a DEVICEID_REGISTRATION
    op whose payload can carry:

    -  Fault-zone coordinates (building, floor, room, rack,
       power domain, network domain, cooling domain).  An
       admin who needs to power down a rack can drive the MDS
       to recall all layouts referencing DSes in that zone and
       evacuate files via CB_PROXY_MOVE before the outage.

    -  Storage media type (SSD / HDD / tape / cloud tier), for
       layout-policy decisions.

    -  Geographic location, for data-locality policy.

    -  Transport security profile (TLS-capable, required
       mutual-TLS cert fingerprint).

    -  Performance tier labels, for admin-assigned QoS.

    -  Encryption-at-rest and compression-at-rest flags.

    -  Scheduled maintenance windows, so the MDS can
       preemptively drain a DS before a planned outage.

    Under this framing, PROXY_REGISTRATION is one arm of a
    generic DEVICEID_REGISTRATION op: the proxy-capability
    arm.  If the WG prefers the generalization, the op in this
    document re-homes as a specialization of
    DEVICEID_REGISTRATION, keeping its wire shape for the
    proxy arm and adding typed entries for the other
    capability classes.  The broader op may land in the main
    draft, in a dedicated draft, or as an extension of this
    document; settlement of that scoping question is the open
    item.

    The op direction (DS -> MDS) is the same for both
    specialized PROXY_REGISTRATION and generalized
    DEVICEID_REGISTRATION; that direction does not today
    exist as a session in the main draft's tight-coupling
    control plane (which runs MDS -> DS).  A resolution of
    this item also settles whether the data-mover draft
    introduces a new DS-initiated session or whether the
    generalized version does.

# Deferred

The items below are explicit protocol extensions identified
during design that this revision does not specify.  They
overlap with Out of Scope in {{sec-scope-out}}; where Out of
Scope frames a deferral in the context of what the mechanism
does do, this list reads as a standalone punch list of
candidate follow-on work items, useful to a future revision's
planner.  A future editorial pass MAY merge this list into
Out of Scope before submission.

-  Partial-range CB_PROXY_MOVE.
-  Multi-proxy pipelines for very large files.
-  Automated proxy selection with load balancing.
-  Proxy-failure predicate (when should the MDS pre-emptively
   replace a slow proxy?).
-  Integration with server-side copy ({{RFC7862}} Section 4)
   as an alternative for single-file moves within one
   namespace.
-  Delta-journaling during a move for online moves without
   dual-writes.
-  FH-stamped affinity tokens (not proposed; deviceinfo
   already carries the necessary information for client-side
   locality shortcuts).

--- back

# Acknowledgments
{:numbered="false"}

David Flynn and Trond Myklebust shaped the data-mover
architecture, in particular the split between proxy
registration and MDS-issued directives.

Brian Pawlowski and Gorry Fairhurst guided this process.
