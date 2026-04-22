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
architecture with a "data mover" role: a registered proxy that
the metadata server directs, via dedicated MDS-to-proxy
operations, to move a file from one layout to another or to
reconstruct a whole file from surviving shards.  The same
mechanism provides codec translation for clients that cannot
participate in a file's native erasure code, including NFSv3
clients.

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

Three classes of work are outside the per-chunk repair model:

1.  **Whole-file repair**, when enough data servers have failed
    that per-chunk reconstruction would require visiting every
    chunk, or when no live client is available to drive the
    repair.

2.  **Layout transitions**, when a file must move from one
    layout geometry to another -- for policy reasons (migrating
    to a new coding type, re-mirroring), for maintenance reasons
    (data-server evacuation), or for environmental reasons
    (migrating between transport-security profiles or between
    filehandle backends).

3.  **Codec translation**, when a client that cannot participate
    in the file's native codec -- including NFSv3 clients --
    still needs to read and write the file.

This document specifies a **data mover** role to address those
three cases with a single mechanism: a proxy that registers with
the metadata server, receives MDS-initiated directives on a
dedicated control session, and provides the server-side of a
client-facing data path during the operation.  Clients discover
the proxy through the normal pNFS layout path -- a layout that
names the proxy with a new data-server flag
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

## In Scope

-  A **registered proxy** role that a data server (or other
   co-located entity) can assume.
-  An op **PROXY_REGISTRATION** by which a proxy declares its
   capabilities and endpoint to the MDS.
-  Two MDS-to-proxy directive ops on the tight-coupling control
   session:
   -  **PROXY_MOVE** -- move a file from a source layout to a
      destination layout.
   -  **PROXY_REPAIR** -- a specialization of PROXY_MOVE where
      the source layout is degraded and the destination layout
      is the reconstructed geometry.
-  The layout conventions clients see during a proxy operation,
   and how clients discover the proxy.
-  **Codec translation for codec-ignorant clients**, including
   NFSv3 clients.  The same proxy machinery that handles move
   and repair also provides the persistent-per-client
   translation that lets a client which cannot participate in a
   file's native codec still read and write the file.  Unlike
   move and repair (which are transient transitions on a file),
   codec translation is an ongoing routing arrangement that
   persists as long as the codec-ignorant client is active.
-  Co-residency attestation ("affinity matching") between a
   client and a local proxy via the PROXY_REGISTRATION affinity
   token.
-  **Credential forwarding rules** for proxies that translate
   on behalf of clients.  The proxy is a translator, not an
   authority; authorization decisions MUST remain with the
   MDS, using the client's forwarded credentials.  See the
   Security Considerations section.
-  Recovery semantics for proxy / MDS / DS failures during a
   proxy operation.

## Out of Scope

-  Journaled delta capture during a move.  A PROXY_MOVE in this
   revision is either quiesced (clients recalled, new layout
   issued after the move completes) or dual-written (proxy
   replicates writes to source and destination while active).
   Delta-journaling mechanisms that allow a proxy to capture
   writes against an otherwise-offline source, replay them on
   completion, or maintain reference integrity across detached
   clones are a future extension.

-  Partial-range PROXY_MOVE (moving a byte range while the
   rest of the file stays on the source).  PROXY_MOVE in this
   revision is always whole-file.

-  Proxy pipelines (multi-proxy staged moves for very large
   files).

-  Automated load balancing or predictive selection across
   registered proxies.

-  Integration with server-side copy ({{RFC7862}} Section 4)
   as an alternative to PROXY_MOVE for single-file moves within
   one namespace.

-  Protocol support for proxy-side features that are
   implementation-internal and do not surface on the wire:
   content-integrity and error-correction layers, encryption
   pass-through, compression pass-through, log-structured
   write staging, sector-alignment normalization, and
   POSIX-loopback shortcuts when proxy and client are
   co-resident.  These are useful motivating scenarios for the
   PROXY_MOVE op but do not require new protocol surface
   beyond what PROXY_MOVE already provides.  A proxy MAY
   implement any or all of them.

# Use Cases

Six motivating scenarios converge on the same mechanism.  In
each case a registered proxy becomes the source of truth for a
file's data during a transition, and clients are redirected to
route I/O through that proxy rather than directly to the
original layout's data servers.

## Administrative Ingest

An administrator rsyncs a file from an external source into the
cluster as a single-copy file.  Server policy requires the file
to be mirrored or erasure coded.  The MDS issues PROXY_MOVE to
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
level.  PROXY_MOVE carries the new layout's geometry and
coding type; the proxy reshapes the file's shards to match.
Because the transformation type (encode / decode / transcode)
is entirely specified by the (source, destination) layout pair,
the op does not need a separate transformation-class field.

## DS Maintenance / Evacuation

A data server is scheduled for maintenance (hardware
replacement, software upgrade, decommission).  All files whose
layouts reference that DS must be evacuated to replacement
DSes before it is taken offline.  PROXY_MOVE is issued against
each file, with source layouts pointing at the outgoing DS and
destination layouts pointing at the replacement.  Evacuation
can be large-scale (thousands or millions of files); running
per-client per-chunk repair over every file would be
prohibitively expensive, but a single registered proxy can
drive many concurrent PROXY_MOVE operations.

## Whole-File Repair

Multiple DSes have failed such that per-chunk repair cannot
reconstruct the file in place.  The MDS constructs a new layout
backed by replacement DSes and issues PROXY_REPAIR to a proxy,
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
PROXY_MOVE applies: the destination DSes have the required
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

This case also covers:

-  NFSv3-to-NFSv4.2 DS protocol upgrade (DS FHs change when the
   DS migrates from RFC 1813 to RFC 8881 semantics).
-  DS-side format change that invalidates existing FHs
   (e.g., transition from local POSIX store to object store).
-  Backend-opaque FH migration where the DS's FH structure is
   internally versioned and old clients hold stale versions.

## Codec Translation for Codec-Ignorant Clients

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

A translating proxy runs two sides:

1.  **Client-facing**: the protocol the codec-ignorant client
    can speak.  For an NFSv3 client this is an NFSv3 server
    that re-exports the MDS's namespace.  For a legacy
    NFSv4.2 client that understands only some codecs, this is
    an NFSv4.2 data-server surface presenting
    FFV2_CODING_MIRRORED (or an equivalent codec the client
    supports).

2.  **MDS-facing**: an NFSv4.2 client to the MDS, plus
    whatever DS protocol the MDS's real DSes speak.  The
    proxy translates each client-facing op into the
    corresponding MDS / DS ops, applies the codec
    transformation, and returns results.

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

The registered-proxy mechanism gives the MDS the information
it needs for translation-proxy selection: `prr_codecs`
enumerates the codecs the proxy can translate between, the
control session carries MDS directives to the proxy, and the
lease bounds the relationship.  No new op is required for the
translation case -- the existing `PROXY_REGISTRATION` covers
it.  `PROXY_MOVE` and `PROXY_REPAIR` are not used for pure
translation (the file is not moving or being repaired); the
proxy simply serves the codec-ignorant client's I/O requests
against the unchanged source layout.

# Design Model

A proxy is a persistent, registered peer of the MDS.  At
startup, a proxy calls **PROXY_REGISTRATION** on the MDS to
declare its capabilities and endpoint.  The MDS records the
registration and MAY select that proxy for subsequent
operations.

When the MDS decides to move or repair a file, it selects a
registered proxy whose capabilities match the operation and
sends it a **PROXY_MOVE** or **PROXY_REPAIR** directive on the
MDS-to-DS tight-coupling control session.  The directive
carries the full specification of the operation: source
layout, destination layout, scheduling parameters.  The proxy
drives the operation to completion and reports progress and
termination back to the MDS.

Clients interact with the proxy through the normal layout
path: when a file is under a proxy operation, the layout the
MDS hands out to clients names the proxy as a data server
(with a dedicated flag), and the client routes CHUNK I/O to
it.  Clients do not receive a special callback for this; the
normal LAYOUTGET / LAYOUTRECALL path carries everything they
need.  Late-arriving clients see the proxy layout from the
start; clients that held an older layout are recalled via
CB_LAYOUTRECALL and reacquire.

This is a deliberate inversion of an earlier CB_PROXY-driven
design: the proxy is the registered server side of the
conversation, not a callback target.  The control flow is
MDS->proxy, not MDS->client.  This matches the
TRUST_STATEID / REVOKE_STATEID pattern in
{{I-D.haynes-nfsv4-flexfiles-v2}} -- a dedicated MDS-to-DS
control session carries MDS authority to the DS, and the DS
acts on it.

# New NFSv4.2 Operations {#sec-new-ops}

This document defines three new NFSv4.2 operations that the
metadata server uses together with a registered proxy.  All
three flow on the MDS-to-proxy control session defined by
{{I-D.haynes-nfsv4-flexfiles-v2}}.  None of them are sent by
pNFS clients.

PROXY_REGISTRATION (91) is issued by a proxy to the metadata
server at startup (and on renewal).  PROXY_MOVE (92) and
PROXY_REPAIR (93) are issued by the metadata server to a
registered proxy.  PROXY_PROGRESS (94) is issued by a proxy to
the metadata server to report the status of an in-flight
PROXY_MOVE or PROXY_REPAIR.

~~~ xdr
/// /* New operations for the Data Mover */
///
/// OP_PROXY_REGISTRATION   = 91;
/// OP_PROXY_MOVE           = 92;
/// OP_PROXY_REPAIR         = 93;
/// OP_PROXY_PROGRESS       = 94;
~~~
{: #fig-data-mover-opnums title="Data Mover operation numbers"}

Opcodes 91 through 94 are chosen to continue the MDS-to-DS
control-plane range that {{I-D.haynes-nfsv4-flexfiles-v2}}
opens at 88 (TRUST_STATEID through BULK_REVOKE_STATEID at
88-90).  If other in-flight NFSv4.2 extensions collide on these
values during IANA coordination, the final assignment will be
reconciled by the consuming RFC editor.

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
/// case OP_PROXY_MOVE: PROXY_MOVE4args opproxymove;
/// case OP_PROXY_REPAIR: PROXY_REPAIR4args opproxyrepair;
/// case OP_PROXY_PROGRESS: PROXY_PROGRESS4args opproxyprogress;
~~~
{: #fig-nfs_argop4-amend title="nfs_argop4 amendment block"}

~~~ xdr
/// /* nfs_resop4 amendment block */
///
/// case OP_PROXY_REGISTRATION:
///     PROXY_REGISTRATION4res opproxyregistration;
/// case OP_PROXY_MOVE: PROXY_MOVE4res opproxymove;
/// case OP_PROXY_REPAIR: PROXY_REPAIR4res opproxyrepair;
/// case OP_PROXY_PROGRESS: PROXY_PROGRESS4res opproxyprogress;
~~~
{: #fig-nfs_resop4-amend title="nfs_resop4 amendment block"}

## Operation 91: PROXY_REGISTRATION - Register as Data Mover {#sec-PROXY_REGISTRATION}

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

A data server (or other co-located entity) calls
PROXY_REGISTRATION on the MDS-to-proxy control session to
declare its capabilities and endpoint.  The MDS records the
registration and MAY select that proxy for subsequent
PROXY_MOVE and PROXY_REPAIR directives.

The prr_codecs field lists the ffv2_coding_type4 values the
proxy supports.  The proxy MUST be able to encode, decode, and
transcode between any pair of values in this list.  Because
the transformation class of a PROXY_MOVE is inherent in the
(source, destination) layout pair, this codec-set membership
is all the capability information the MDS needs to match.  An
empty list results in NFS4ERR_INVAL in this revision.

The prr_affinity field is an opaque token the proxy supplies
for co-residency attestation with a client.  The MDS does not
interpret it.  At layout-grant time the MDS compares
prr_affinity against the requesting client's co_ownerid
(Section 18.35 of {{RFC8881}}); a match indicates that the
client and proxy share a host, and the MDS MAY use it as input
to proxy selection.  See {{sec-affinity}} for the matching
algorithm.  An empty prr_affinity means the proxy does not
claim co-residency with any client.

The prr_lease field is the lease duration the proxy requests
in seconds.  The MDS MAY grant a shorter one, returned in
prr_granted_lease.  The proxy MUST renew before the granted
lease expires; on expiry the MDS drops the registration and
any in-flight PROXY_MOVE or PROXY_REPAIR is reassigned.

The prr_flags field is reserved for future use.  It MUST be
set to 0 in this revision.

On success, the MDS returns a prr_registration_id that
identifies this registration.  The proxy uses it to renew the
registration and to correlate PROXY_MOVE and PROXY_REPAIR
directives the MDS subsequently issues.

Registration conveys capabilities only; the proxy's network
endpoint is conveyed through the same deviceinfo channel as
any other DS's address.  When the MDS selects a proxy for an
operation, the layout issued to clients includes a
ffv2_data_server4 entry pointing at the proxy's existing
deviceinfo.

PROXY_REGISTRATION is issued on the dedicated MDS-to-proxy
control session (per the Tight Coupling Control Session
subsection of {{I-D.haynes-nfsv4-flexfiles-v2}}).  The proxy
MUST present EXCHGID4_FLAG_USE_PNFS_MDS on that session so
the MDS can gate the PROXY_* ops the same way it gates
TRUST_STATEID.

## Operation 92: PROXY_MOVE - Direct a Registered Proxy to Move a File {#sec-PROXY_MOVE}

### ARGUMENTS

~~~ xdr
/// const PMA_FLAG_DUAL_WRITE = 0x00000001;
///
/// struct PROXY_MOVE4args {
///     uint64_t         pma_registration_id;
///     nfs_fh4          pma_fh;
///     ffv2_layout4     pma_source_layout;
///     ffv2_layout4     pma_destination_layout;
///     nfstime4         pma_deadline;
///     uint32_t         pma_flags;
/// };
~~~
{: #fig-PROXY_MOVE4args title="XDR for PROXY_MOVE4args"}

### RESULTS

~~~ xdr
/// struct PROXY_MOVE4resok {
///     uint64_t         pmr_operation_id;
/// };
///
/// union PROXY_MOVE4res switch (nfsstat4 pmr_status) {
///     case NFS4_OK:
///         PROXY_MOVE4resok  pmr_resok4;
///     default:
///         void;
/// };
~~~
{: #fig-PROXY_MOVE4res title="XDR for PROXY_MOVE4res"}

### DESCRIPTION

The metadata server issues PROXY_MOVE to a registered proxy to
direct it to move a file from one layout to another.  The move
covers the whole file; partial-range moves are out of scope for
this revision.

The pma_registration_id field MUST reference an active
registration (an unexpired prr_registration_id returned by a
prior PROXY_REGISTRATION).  An expired or unknown registration
MUST be rejected.

The pma_fh field is the MDS filehandle of the file to move.

The pma_source_layout and pma_destination_layout fields are
full layout specifications.  The proxy reads from the source
layout's DSes and writes to the destination layout's DSes.
Either layout MAY reference DSes not otherwise participating
in the operation; the proxy acts as an NFSv4.2 client to each
DS named in either layout.  The transformation class (encode,
decode, transcode, or identity-with-relocation) is determined
entirely by the (source, destination) pair; no separate
transformation field is needed.

The pma_deadline field is a wall-clock nfstime4 by which the
proxy is expected to have completed the move.  Missing the
deadline does not corrupt state, but the MDS MAY cancel and
reassign.

The pma_flags field currently defines one flag:

-  PMA_FLAG_DUAL_WRITE: during the move the proxy accepts
   client writes and replicates them to both source and
   destination mirror sets.  On completion the source is
   retired without rollback risk.  Used for online transitions
   where clients continue writing.

   When unset, the move is quiesced: the MDS recalls client
   layouts for the duration, clients that open during the move
   are directed through the proxy, and the proxy does not
   replicate to the source because no client is writing.

On success the proxy returns a proxy-assigned pmr_operation_id
for the in-flight move, used for progress queries and
cancellation.

The proxy reports progress by calling PROXY_PROGRESS (see
{{sec-PROXY_PROGRESS}}) on the same control session.
PROXY_PROGRESS MAY be issued at any cadence for interim
updates and MUST be issued exactly once with ppa_terminal=true
when the operation reaches a terminal state.  On receipt of a
terminal PROXY_PROGRESS with ppa_status=NFS4_OK, the MDS
issues CB_LAYOUTRECALL to any client still on the old layout,
waits for LAYOUTRETURNs, and retires the source DSes.

Terminal outcomes (communicated via ppa_status in the terminal
PROXY_PROGRESS, not as the return code of PROXY_MOVE itself):

-  NFS4_OK: destination fully populated and consistent.
-  NFS4ERR_PAYLOAD_LOST: source layout degraded beyond
   reconstructibility; operation aborted and the MDS marks
   affected byte ranges lost.
-  Other codes: proxy-specific failure; the MDS MAY retry or
   reassign.

NFS4ERR_DELAY appears only in interim (non-terminal)
PROXY_PROGRESS updates to signal that the proxy needs more
time; it MUST NOT be used as a terminal status.

## Operation 93: PROXY_REPAIR - Direct a Registered Proxy to Reconstruct a File {#sec-PROXY_REPAIR}

### ARGUMENTS

~~~ xdr
/// struct PROXY_REPAIR4args {
///     uint64_t         pra_registration_id;
///     nfs_fh4          pra_fh;
///     ffv2_layout4     pra_source_layout;
///     ffv2_layout4     pra_destination_layout;
///     nfstime4         pra_deadline;
/// };
~~~
{: #fig-PROXY_REPAIR4args title="XDR for PROXY_REPAIR4args"}

### RESULTS

~~~ xdr
/// struct PROXY_REPAIR4resok {
///     uint64_t         prer_operation_id;
/// };
///
/// union PROXY_REPAIR4res switch (nfsstat4 prer_status) {
///     case NFS4_OK:
///         PROXY_REPAIR4resok  prer_resok4;
///     default:
///         void;
/// };
~~~
{: #fig-PROXY_REPAIR4res title="XDR for PROXY_REPAIR4res"}

### DESCRIPTION

PROXY_REPAIR is a convenience specialization of PROXY_MOVE
where the source layout is degraded.  The wire shape is nearly
identical to PROXY_MOVE; the separate op exists for
readability (MDS intent is apparent from the op name) and to
reserve the option of repair-specific progress semantics.

The argument fields are analogous to the corresponding fields
in PROXY_MOVE4args, with pra_source_layout understood to
describe a degraded layout -- one or more DSes in the source
layout are unreachable or hold shards that cannot be validated.
The proxy reconstructs the file's data from whatever surviving
shards the source layout references and writes the
reconstructed content to the destination layout.  If fewer
than k shards survive across the mirror set, the operation
terminates with NFS4ERR_PAYLOAD_LOST for the affected byte
ranges, matching the per-chunk repair semantics in the Repair
Client Selection section of
{{I-D.haynes-nfsv4-flexfiles-v2}}.

An implementation MAY collapse PROXY_MOVE and PROXY_REPAIR
into a single internal dispatch.  PROXY_REPAIR is always
quiesced (no PMA_FLAG_DUAL_WRITE equivalent); client writes
cannot be safely replicated until the destination is
consistent.

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

A registered proxy calls PROXY_PROGRESS on the MDS-to-proxy
control session to report the status of an in-flight
PROXY_MOVE or PROXY_REPAIR.  The proxy MAY send progress
updates at any cadence it chooses.  The MDS uses non-terminal
updates for observability and liveness detection; it MUST NOT
require any specific update cadence as a correctness condition.

The ppa_registration_id field identifies the proxy's
registration.  The ppa_operation_id field identifies the
specific PROXY_MOVE or PROXY_REPAIR operation being reported
on; it is the value the proxy returned in pmr_operation_id or
prer_operation_id when the directive was issued.

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

   -  NFS4ERR_PAYLOAD_LOST: for a PROXY_REPAIR, the source
      layout was degraded beyond reconstructibility and the
      MDS MUST mark the affected byte ranges lost.  For a
      PROXY_MOVE, a catastrophic loss of both source and
      destination during the move that cannot be recovered.

   -  Any other nfsstat4: proxy-specific failure.  The MDS MAY
      retry by reassigning the operation to a different
      registered proxy (subject to the original deadline and
      any retry policy the MDS applies).

The ppa_bytes_done and ppa_bytes_total fields are advisory.
When the proxy knows both (for example, moves on files with a
stable size), it SHOULD populate them.  When the total is not
known (for example, a source that is still being appended to),
the proxy MUST set ppa_bytes_total to 0; ppa_bytes_done MAY
still carry cumulative progress.

The MDS returns ppr_status of NFS4_OK to acknowledge a
well-formed PROXY_PROGRESS, or an error if the registration
or operation id is unknown.  For terminal updates, the proxy
MUST retry if it does not receive an acknowledgment within an
implementation-defined timeout; the MDS MUST treat duplicate
terminal updates for the same (ppa_registration_id,
ppa_operation_id) pair as idempotent.  Non-terminal updates
are fire-and-forget and need not be retried.

Cancellation of an in-flight operation by the MDS is out of
scope for this revision; see {{sec-open-questions}}.

# Affinity Matching {#sec-affinity}

A proxy MAY populate prr_affinity at registration time with a
token that lets the MDS recognize co-residency with a client.
The canonical use case: a client host is running a local proxy,
potentially in a container on the same physical machine as the
client.  Network-level checks alone are not reliable -- the
proxy may be on a container IP, a separate netns, or behind a
NAT -- so an explicit token is required.

The matching algorithm:

1.  When a client issues EXCHANGE_ID to the MDS, the client's
    co_ownerid MAY contain an affinity token.  Clients that do
    not wish to participate send a normal co_ownerid.

2.  When the MDS processes a LAYOUTGET on a file that has an
    active PROXY_MOVE / PROXY_REPAIR, it iterates over
    registered proxies and compares each proxy's prr_affinity
    against the requesting client's co_ownerid.  A match
    (equality, substring, or hash equivalence -- the match
    predicate is implementation-defined but MUST be
    deterministic for a given MDS instance) indicates
    co-residency.

3.  The MDS MAY use a match as input to proxy selection.  When
    multiple proxies are eligible, the MDS MAY prefer the one
    matching the most clients' affinity tokens.

4.  The layout returned names the selected proxy first in
    ffs_data_servers.  For mirrored coding types this makes
    the local proxy the client's default read target (matching
    the FFv1 affinity pattern).  For erasure-coded types it
    makes the local proxy the preferred encode endpoint.

Affinity is advisory.  The MDS MUST NOT grant any authority
based solely on affinity; the normal authentication model
still applies, and a client that claims an affinity token it
has no right to gains at most a sub-optimal layout, not
unauthorized access.

Note: there is no proposal to stamp the affinity token into the
filehandle.  The MDS has already performed the match before the
layout reaches the client, and the layout's deviceinfo carries
all the information a client-side shortcut needs.

# Layout Shape During a Proxy Operation

When a PROXY_MOVE or PROXY_REPAIR is active for a file, the
layout the MDS hands out to clients contains:

-  The **proxy DS entry** at the head of ffs_data_servers (or
   otherwise flagged for routing; a new flag is defined below).
   This entry names the selected proxy.

-  Optional source and destination DS entries, visible or
   hidden to the client at the MDS's discretion.  In the
   simplest case the proxy is the only visible DS; source and
   destination mirror sets are internal to the proxy.

-  A new flag on ffv2_ds_flags4:

~~~
const FFV2_DS_FLAGS_PROXY = 0x00000040;  // TBD, IANA alloc
~~~

When FFV2_DS_FLAGS_PROXY is set on any data server entry in a
layout, clients MUST direct all CHUNK I/O for this file to that
DS rather than to any other data server in the layout.  The
proxy DS internally dispatches reads and writes to the source
and destination DSes.

## Single-Layout Model

This design uses a single layout with a PROXY-flagged entry,
not two linked layouts.  Rationale:

-  pNFS clients already handle single layouts cleanly; no new
   layout-linkage mechanism is needed.
-  The client's view ("the file's DS is the proxy") is the
   truth during the operation.  Exposing the source and
   destination DSes directly would invite confusion.
-  Late-arriving clients see the proxy layout from the start;
   no separate path for them.

# Client Behavior

A client that observes a layout with FFV2_DS_FLAGS_PROXY:

1.  Routes all CHUNK I/O to the PROXY-flagged DS.

2.  Does not issue I/O directly to any non-PROXY DS in the
    layout.  Non-PROXY DSes MAY appear in the layout for
    informational reasons but MUST NOT be addressed by the
    client.

3.  Uses its existing layout stateid against the proxy DS.
    The proxy accepts CHUNK ops under that stateid (the MDS
    has registered the stateid via TRUST_STATEID on the proxy
    per main-draft tight-coupling semantics).

4.  Handles proxy-side errors (NFS4ERR_DELAY, connection loss,
    NFS4ERR_BAD_STATEID) exactly as it would any other DS
    error: LAYOUTERROR to the MDS and expect a new layout or
    proxy reassignment.

## When the Layout Is Recalled

If the MDS recalls the layout mid-operation (the proxy failed
and is being replaced, or the operation completed and normal DS
layouts are being reissued), the client LAYOUTRETURNs as usual
and reacquires via LAYOUTGET.  The new layout may have a
different proxy, a different proxy set, or no proxy entry if
the operation has completed.

## In-Flight I/O When the Proxy Changes

In-flight I/O to the old proxy when the MDS recalls the layout
MAY complete at the old proxy; results remain valid under the
old proxy's authority.  New I/O issued after LAYOUTRETURN MUST
go through the replacement proxy (or, if the new layout has no
proxy, directly to the DSes named there).

# State Machine

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
                               | MDS selects registered proxy,
                               | issues PROXY_MOVE (or
                               | PROXY_REPAIR) on control
                               | session
                               v
                         +--------------+
                         | PROXY_ACTIVE |
                         | clients see  |
                         | layout with  |
                         | proxy DS at  |
                         | head of      |
                         | ffs_data_    |
                         | servers;     |
                         | proxy drives |
                         | source->dest |
                         +-----+--------+
                               |
                               | proxy reports completion
                               | (destination fully populated
                               | and consistent)
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
| READY | PROXY_ACTIVE | MDS decides to move/repair | MDS issues PROXY_MOVE or PROXY_REPAIR on control session; MDS starts handing out proxy layouts |
| PROXY_ACTIVE | COMMITTING | Proxy reports completion | MDS begins CB_LAYOUTRECALL fan-out |
| COMMITTING | DONE | All clients LAYOUTRETURNed | MDS issues post-move layouts; source DSes retired |
| PROXY_ACTIVE | READY | Proxy failed, no replacement | MDS cancels; layouts revert to pre-move source set |

# Proxy Failure and Recovery

## Proxy Crash During PROXY_ACTIVE

1.  Client I/O to the proxy receives NFS4ERR_DELAY (if the
    proxy is reachable but unhealthy) or connection errors (if
    unreachable).  Clients report LAYOUTERROR.

2.  The MDS MAY select a replacement proxy from the registered
    pool and issue PROXY_MOVE / PROXY_REPAIR to the
    replacement, with the source layout updated to reflect
    current reality (destination DSes that the failed proxy
    populated are now part of the source set) and the
    destination layout unchanged.  The replacement proxy
    resumes from wherever the failed proxy left off.

3.  The MDS issues CB_LAYOUTRECALL on the old layout and the
    replacement proxy's layout becomes live for new LAYOUTGETs.

4.  If the MDS cannot find a replacement within a policy
    timeout, it MUST cancel the operation: revert to the
    pre-move source layout, do not issue a destination layout,
    mark the destination DSes for cleanup or retry.

## Cascading Proxy Failure

Repeated proxy failures on the same operation SHOULD trigger
escalation to deployment management rather than recursive
retry.  Recurring failures likely indicate an environmental
issue the proxy cannot work around.

## Source DS Crash During PROXY_ACTIVE

Reduces the proxy's read parallelism but does not block forward
progress as long as the erasure code can still reconstruct
from surviving source DSes.  If the source degrades past
reconstructibility, the operation transitions to whole-file
repair semantics automatically: partial reconstruction
succeeds; ranges that cannot be reconstructed terminate the
operation with NFS4ERR_PAYLOAD_LOST.

## Destination DS Crash During PROXY_ACTIVE

Treated as a normal DS failure on the destination side.  The
proxy acts like a client to the destination DSes: LAYOUTERROR
to the MDS, which MAY substitute a spare or mark the
destination FFV2_DS_FLAGS_REPAIR.  The proxy continues pushing
to the remaining destinations.  Clients are unaffected.

# MDS Crash Recovery

1.  Clients and the proxy detect MDS session loss and enter
    RECLAIM per {{RFC8881}}.

2.  The proxy reclaims its PROXY_REGISTRATION.  The MDS MAY
    persist registrations across restart, or the proxy MAY
    re-register; PROXY_REGISTRATION is idempotent as long as
    prr_registration_id is preserved by the proxy.

3.  The MDS re-advertises proxy layouts (with the PROXY flag)
    to reclaiming clients.

4.  If the MDS persisted its view of the active PROXY_MOVE /
    PROXY_REPAIR operations (which destination DSes are
    populated, which chunks are committed, etc.), the proxy
    resumes from the last persisted checkpoint.  Persistence
    is RECOMMENDED but not required; a non-persistent MDS
    restarts the operation from the beginning.

5.  If the MDS crashed between the proxy reporting completion
    and the MDS issuing CB_LAYOUTRECALL, the MDS on restart
    SHOULD re-drive the COMMITTING phase.

# Backward Compatibility

## Clients

Client behavior is a normal layout path with a new flag.
Clients that do not recognize FFV2_DS_FLAGS_PROXY will treat
the proxy DS as any other DS and route I/O to it normally;
that is in fact the correct behavior.  No client-side version
negotiation is needed.

Clients that require strict per-DS identity checking (e.g.,
"the layout's DS must match a pre-allowlisted fingerprint")
should extend their allowlist to include registered proxies.
This is a deployment concern, not a protocol one.

## Data Servers

DSes without proxy capability simply do not call
PROXY_REGISTRATION and are never selected as proxies.  A
deployment with no registered proxies falls back to:

-  Per-chunk CB_CHUNK_REPAIR for single-shard repair.
-  Admin-coordinated offline procedures for policy transitions
   and DS evacuation.
-  Blocking DS maintenance (the DS cannot drain through a
   proxy).

Deployments SHOULD ensure at least one registered proxy exists
per failure domain to avoid a single point of failure on move
operations.

## NFSv3 Source DSes

When the source mirror is an NFSv3 DS, the proxy reads from it
using NFSv3 semantics and writes to the NFSv4.2 destination
using CHUNK semantics.  This is the same pattern the main
draft uses for InBand I/O.

# Security Considerations

1.  **Proxy authority.**  A proxy in PROXY_ACTIVE sees all
    client I/O for the proxied file.  A compromised proxy can
    observe or modify file data.  Deployments MUST treat
    proxy-capable DSes as at least as trusted as the DSes
    they proxy for.  PROXY_REGISTRATION SHOULD be gated by a
    deployment-level allowlist; arbitrary DSes that present
    the op without prior provisioning SHOULD be rejected.

2.  **Transport security across the operation.**  The proxy's
    connections to source and destination DSes are independent
    of the client's connection to the proxy.  A proxy MAY read
    from an AUTH_SYS source and write to a TLS destination (or
    any other combination).  The proxy is responsible for
    enforcing the effective security policy (e.g., do not
    downgrade encrypted data to a plaintext DS).

3.  **Principal binding during a proxy operation.**  For
    proxy-to-DS traffic (the proxy reading source DSes and
    writing destination DSes to carry out a PROXY_MOVE or
    PROXY_REPAIR), the proxy presents a principal to those
    DSes that they will accept; this is the proxy's own
    service identity unless constrained delegation or
    equivalent is arranged.  Forwarding the client's identity
    to the peer DSes for proxy-driven data movement is NOT
    required and is typically NOT practical (the client is
    not in the conversation at that point).  See however the
    Credential Forwarding and Privilege Boundary section
    below for the case of client-initiated file I/O through a
    translating proxy, where the credential-forwarding rule
    is different and stricter.

4.  **Proxy impersonation.**  A malicious MDS could register a
    hostile entity as a proxy.  The existing MDS trust model
    already grants the MDS this capability via
    CB_LAYOUTRECALL + new layout; PROXY_REGISTRATION does not
    weaken it.  Clients that require stronger proxy identity
    verification SHOULD validate the proxy's transport-
    security credentials against a deployment allowlist.

5.  **Affinity token.**  prr_affinity is a co-residency
    attestation, not an authentication mechanism.  Matching an
    affinity token between a proxy and a client grants the
    client no new access rights; it is used only as input to
    proxy-selection preferences.  A client cannot elevate
    privilege by spoofing an affinity token.

6.  **Registration lease expiry.**  If a proxy's lease expires
    mid-operation, the MDS MUST cancel the operation (revert
    layouts, mark destination DSes for cleanup).  The MDS
    MUST NOT continue to route client I/O to a proxy whose
    registration has lapsed.

## Credential Forwarding and the Privilege Boundary

A translating proxy (see Codec Translation for Codec-Ignorant
Clients) has structurally elevated privilege by design.  To
perform its management tasks -- moves, repairs, evacuations,
cross-tenant re-exports -- the deployment grants the proxy's
service identity broad access: typically not-root-squashed,
often read/write to every file in the namespace, and session
authority to every DS.  That privilege is intentional.

A codec-ignorant client that reaches the proxy, however,
arrives with its own RPC credentials that the proxy does not
itself need in order to function.  An NFSv3 client's uid/gid,
an AUTH_SYS-squashed identity, an RPCSEC_GSS principal -- none
of these are the proxy's own.  If the proxy ignores the
client's credentials and issues MDS / DS operations under its
own service identity when translating client I/O, every
client that reaches the proxy silently inherits the proxy's
privilege.  This is a protocol-level privilege-escalation
vector, and this document calls it out rather than hiding it.

The normative requirements below apply whenever a proxy is
translating client-initiated file I/O (as distinct from
proxy-driven move / repair work, which runs under the proxy's
own authority on directives from the MDS).

1.  **Credential pass-through.**  The proxy MUST present the
    client's credentials (RPC auth flavor and principal) on
    every MDS or DS operation it issues as a consequence of a
    client-initiated request.  Specifically, a client `READ`
    that the proxy expands into `LAYOUTGET` + `CHUNK_READ`
    MUST carry the client's credentials on both the
    `LAYOUTGET` against the MDS and the `CHUNK_READ` against
    the DSes.  The proxy MUST NOT substitute its own service
    identity for client-initiated operations.

2.  **No squash inversion.**  If the client arrives with a
    root-squashed identity (for example, uid 0 mapped to
    nobody by the NFSv3 export configuration on the
    client-facing side of the proxy), the proxy MUST preserve
    the squashed identity when forwarding.  The proxy MUST NOT
    translate a client's squashed credentials back into
    unsquashed root, even though the proxy's own identity is
    typically unsquashed.

3.  **Authorization remains with the MDS.**  The MDS MUST
    perform access-control checks against the forwarded client
    credentials, not against the proxy's service identity, for
    any client-initiated file operation.  The proxy is a
    translator, not an authority.  This is what prevents
    proxy deployment from becoming a blanket ACL override.

4.  **Proxy service identity is for the control plane only.**
    The proxy MAY, and typically MUST, use its own service
    identity for:
    -  The MDS-to-proxy control session (EXCHANGE_ID with
       EXCHGID4_FLAG_USE_PNFS_MDS, the session on which
       PROXY_REGISTRATION and PROXY_MOVE / PROXY_REPAIR
       directives flow).
    -  Peer-DS session setup for proxy-driven data movement
       (reading source DSes, writing destination DSes under
       a PROXY_MOVE that the MDS has authorized).
    -  Proxy housekeeping.

    The proxy's service identity MUST NOT be used for
    client-initiated file data operations.

5.  **Failure mode on missing credentials.**  If the proxy
    cannot forward a client's credentials for some reason
    (e.g., the client presented AUTH_NONE, or the client-facing
    side used a security flavor the proxy cannot propagate),
    the proxy MUST reject the client operation with the
    equivalent of NFS4ERR_ACCESS (or NFS3ERR_ACCES for NFSv3
    clients).  The proxy MUST NOT fall back to serving the
    operation under its own identity.

Deployment-level requirements:

-  PROXY_REGISTRATION MUST be allowlisted.  An unknown DS
   presenting PROXY_REGISTRATION MUST be rejected.  This is
   the only wire-level defense against a hostile entity
   registering as a proxy and then receiving client-forwarded
   credentials.

-  The MDS-to-proxy control session MUST use RPCSEC_GSS
   {{RFC7861}} or RPC-over-TLS {{RFC9289}} with mutual
   authentication.  AUTH_SYS on the control session is
   forbidden.

-  Deployments SHOULD audit both the proxy's credential-
   forwarding behavior (the proxy logs what it forwards) and
   the MDS's authorization checks (the MDS logs what principal
   authorized each operation).  Divergence between the two
   indicates a credential-forwarding bug or compromise.

What the protocol cannot defend against:

-  A compromised proxy has direct access to whatever
   credentials pass through it.  Credential confidentiality
   collapses the moment the proxy is under adversary control.
   Mitigation is operational: restrict which hosts can
   register as proxies, audit PROXY_REGISTRATION events,
   rotate deployment-level keys.

-  A deployment that configures a proxy to run AS root while
   the client is root-squashed has already violated rule 2
   above; no wire mechanism detects a proxy deliberately
   mis-implementing credential forwarding.  Deployments SHOULD
   verify their proxy implementation's credential-forwarding
   behavior through conformance testing before production use.

Future work (noted as an Open Question below): RPCSEC_GSSv3
structured privilege assertion per {{RFC7861}} Section 2.5.2
is the natural strong-authentication mechanism for
proxy-forwarded credentials.  This revision does not require
GSSv3 because the broader NFSv4 deployment base does not yet
support it; deployments that can use GSSv3 SHOULD prefer it
over AUTH_SYS passthrough for the credential-forwarding
channel.

# IANA Considerations {#iana-considerations}

This document does not require any IANA action.

The three new NFSv4.2 operations defined in
{{sec-new-ops}} (OP_PROXY_REGISTRATION = 91, OP_PROXY_MOVE = 92,
OP_PROXY_REPAIR = 93) follow the convention that NFSv4.2
operation numbers are governed by the publishing document and
do not require a separate IANA registry entry.  The same
convention applies to the new flag bit
FFV2_DS_FLAGS_PROXY, which is an additional bit in the
ffv2_ds_flags4 bitmap defined by
{{I-D.haynes-nfsv4-flexfiles-v2}}; that document explicitly
records its flag-word bitmaps as not IANA-registered, and any
future bit allocations are made by a document that updates or
obsoletes it.

The PMA_FLAG_DUAL_WRITE bit in pma_flags (defined in
{{sec-PROXY_MOVE}}) is a bit in a bitmap this document
introduces.  Following the precedent in
{{I-D.haynes-nfsv4-flexfiles-v2}} (which in turn follows
{{RFC8435}}), this document does not establish an IANA
registry for its bit spaces; future bit allocations are made
by a document that updates or obsoletes this one.
Implementations MUST treat unknown bits as reserved and MUST
NOT assign meaning to them locally.

# Interaction with the Main Draft {#interaction}

## chunk_guard4

The proxy enforces chunk_guard4 CAS on the destination mirror
set on behalf of clients.  The proxy MAY use the same guard
values client writes carry through it, or generate fresh guard
values on the destination side, provided uniqueness on the
destination is preserved.

## CHUNK_LOCK

If a client holds a chunk lock on a file when a proxy
operation activates, the lock follows the file: the proxy
takes ownership of the lock on the destination mirror set, and
the MDS-escrow semantics (the Reserved cg_client_id Value
subsection of {{I-D.haynes-nfsv4-flexfiles-v2}}) apply if the
original holder becomes unreachable during the operation.

## CB_CHUNK_REPAIR

Per-chunk CB_CHUNK_REPAIR and a PROXY_MOVE or PROXY_REPAIR on
the same file are mutually exclusive at any given time.  The
MDS MUST NOT issue CB_CHUNK_REPAIR for a file currently in
PROXY_ACTIVE; the proxy handles any mid-move repair
internally.  If the MDS decides a proxied file also needs
per-chunk repair after the proxy operation completes, it
issues CB_CHUNK_REPAIR against the post-move layout.

## TRUST_STATEID / REVOKE_STATEID

When the MDS selects a proxy, it issues TRUST_STATEID on the
proxy for every client layout stateid that will route through
the proxy during PROXY_ACTIVE.  On proxy retirement the MDS
issues REVOKE_STATEID on the retired proxy.  This is the same
mechanism {{I-D.haynes-nfsv4-flexfiles-v2}} defines for any
DS in a tightly coupled deployment.

# Open Questions {#sec-open-questions}

1.  **Cancellation of in-flight operations.**  The design
    specifies PROXY_PROGRESS as the proxy-to-MDS channel for
    reporting status, but does not define an MDS-to-proxy
    cancellation mechanism.  When the MDS decides to abort an
    in-flight PROXY_MOVE or PROXY_REPAIR (for example, because
    a higher-priority operation needs the same proxy, or the
    source layout became unusable before the move started),
    how does it tell the proxy to stop?  Options: a new op
    PROXY_CANCEL; a cancellation flag carried on the MDS
    response to a non-terminal PROXY_PROGRESS; an implicit
    cancellation via REVOKE_STATEID that invalidates the
    proxy's access.  Worth settling before first submission.

2.  **Registration renewal semantics.**  Is renewal a fresh
    PROXY_REGISTRATION with the same prr_registration_id
    (idempotent), or a separate PROXY_RENEW op (lighter-
    weight)?

3.  **Affinity match predicate.**  Is exact equality of
    prr_affinity and co_ownerid sufficient, or does the spec
    need to define substring / hash matching explicitly?  If
    implementation-defined, MDS implementations that pick
    different predicates will produce different layouts --
    is that acceptable?

4.  **Multiple concurrent proxies per file.**  The design
    assumes one proxy per file per operation.  Should two
    proxies be allowed to pipeline a large file (proxy A
    drives the first 1 TB, proxy B drives the next)?  Adds
    state-machine complexity.

5.  **Transitive proxy.**  If a file in PROXY_ACTIVE needs a
    second move (e.g., DS maintenance interrupts a repair),
    what happens?  Queue the second move?  Abort the first?
    Allow a proxy to act as the source for another proxy?

6.  **Persistent vs ephemeral MDS operation state.**  Is
    operation persistence a SHOULD or a MAY?  Production
    deployments probably want SHOULD to avoid restart cost on
    large moves; prototypes probably want MAY.

7.  **Registration as a capability-scoped authority.**  Should
    PROXY_REGISTRATION require a separate EXCHGID4 flag (e.g.,
    EXCHGID4_FLAG_USE_PROXY_DS) to distinguish proxy-capable
    DSes from generic DSes, or is the registration itself the
    capability declaration?

8.  **Richer capability advertising.**  prr_codecs covers the
    transformation classes that matter for move / repair.
    Features that are implementation-internal (encryption,
    compression, alignment normalization) do not need to be
    advertised because they do not affect the wire contract.
    Features that DO affect the wire (e.g., support for some
    future sparse-read or TRIM op) would warrant a richer
    capability descriptor.  Worth revisiting when those ops
    are defined.

9.  **RPCSEC_GSSv3 for translating-proxy credential
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

# Deferred

-  Partial-range PROXY_MOVE.
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
