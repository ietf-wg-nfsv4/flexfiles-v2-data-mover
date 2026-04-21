# Data Mover: Proxy DS Authority for Layout Transitions

## Background

Hammerspace's Flex Files v1 ({{RFC8435}}) deployment includes an internal
mechanism for moving, copying, and repairing file data across data servers
without blocking concurrent client I/O.  The mechanism is not specified in
RFC 8435; it lives under the covers of the Hammerspace server as a
proprietary concept.  Other pNFS implementations have similar needs --
typically handled through their own proprietary control protocols or
administrator-coordinated offline procedures.

This design codifies the mechanism as a standard Flex Files v2 primitive.
Goals:

1.  Multiple implementations can interoperate on the same mover-driven
    workflows -- today this is the single biggest interop gap between
    pNFS deployments and parallel-filesystem competitors such as
    Lustre.
2.  The whole-file repair case described in Section 10.6 of
    draft-haynes-nfsv4-flexfiles-v2 has a well-defined specification
    rather than a pointer to "ad hoc server logic."
3.  Policy-driven and maintenance-driven data movement stop being
    server-internal and become inspectable on the wire.

Relation to the main draft: the main draft defines CB_CHUNK_REPAIR and
the per-chunk repair model (Section sec-repair-selection).  This design
addresses the file-scoped cases that per-chunk repair cannot cover
without substantial overhead: whole-file repair, policy transitions,
DS evacuation, and transport/FH migrations.

## Use Cases

Six motivating scenarios converge on a single mechanism.  In each case,
some non-client entity -- usually a designated DS -- becomes the source
of truth for a file's data during a transition, and clients must be
redirected to route I/O through that entity rather than directly to
the original layout's data servers.

### Administrative Ingest

An admin rsyncs a file from an external source into the cluster as a
single-copy file.  Server policy requires the file to be mirrored or
erasure coded.  The MDS creates a destination layout of the required
shape, assigns a data mover to populate it from the source, and
redirects any client that opens the file during the move to route I/O
through the mover.

The source "layout" may not even be a Flex Files layout -- it could be
a non-pNFS NFS mount that the mover itself reads as an NFSv4.2 client.
The key property is that throughout the move, the mover presents the
file to pNFS clients as if the move had not yet started, while
populating the destination in the background.

### Policy-Driven Layout Transition

A server-objective or policy change (e.g., "files older than 30 days
must be erasure coded", "high-access-rate files must have additional
mirrors") requires transforming a file's layout without user
visibility.  The transformation is purely a layout change; the file
contents are unchanged except at the shard level.

### DS Maintenance / Evacuation

A DS is scheduled for maintenance (hardware replacement, software
upgrade, decommission).  All files whose layouts reference that DS
must be evacuated to replacement DSes before it is taken offline.
Each file needs a mover that copies the DS's contents to the
destination DS(es) while clients continue to read and write.
Evacuation can be large-scale (thousands or millions of files);
running per-client per-chunk repair over every file is prohibitively
expensive.

### Whole-File Repair

Multiple DSes have failed such that per-chunk repair (see
sec-repair-selection in the main draft) cannot reconstruct the file
in place.  The MDS constructs a new layout backed by replacement
DSes, assigns a mover, and drives reconstruction from whatever
surviving shards remain.  If fewer than k shards survive across the
mirror set, the move terminates with NFS4ERR_PAYLOAD_LOST, matching
the per-chunk repair semantics.

### TLS Coverage Transition

A file whose layout currently points at non-TLS-capable data servers
needs to be migrated to TLS-capable data servers, or vice versa
(declassification of previously-encrypted data; a policy change
mandating transport security; onboarding a new storage class whose
DSes are TLS-only).  The mover pattern applies: the destination DSes
have the required transport security profile, the source DSes are
retired.  A client that arrives mid-transition is redirected through
the mover and does not directly see the heterogeneous DS set.

Design implication: the mover MUST be able to establish its own RPC
connections to both source and destination DSes, potentially with
different transport security profiles (non-TLS to source, mutual
TLS {{RFC9289}} to destination, or any other combination).  The
mover's per-DS security is independent of the client's security to
the mover.

### Filehandle / Storage-Backend Transition

A DS changes the filehandles it issues for a file -- this happens
when the DS's underlying storage is migrated (e.g., from one backend
object store to another) and the old filehandles become unresolvable
on the new backend.  Without a mover, every client holding a layout
must be individually recalled and re-issued; with a mover, the MDS
points all clients at the mover (keeping their existing stateids and
FHs intact), the mover reconciles old-to-new FHs internally, and
clients are recalled only at the end of the move.

This case also covers:

-  NFSv3-to-NFSv4.2 DS protocol upgrade (FHs change when the DS
   migrates from RFC 1813 to RFC 8881 semantics).
-  DS-side format change that invalidates existing FHs (e.g.,
   transition from local POSIX store to object store).
-  Backend-opaque FH migration where the DS's FH structure is
   internally versioned and old clients hold stale versions.

## Proposal: Data Mover Role

A **data mover** is a role that a data server can assume for a file
during a layout transition.  When a DS is acting as a data mover for
a file, it:

1.  Accepts client CHUNK ops (and NFSv3 READ / WRITE / COMMIT, where
    applicable) for that file, instead of the client I/O going
    directly to the mirror-set DSes.

2.  Is authoritative for the file's data during the transition: the
    mover's view of each chunk is the source of truth, even when
    source-layout DSes still have chunks.

3.  Performs the actual data movement to the destination layout's
    DSes by acting as an NFSv4.2 client to those DSes (or NFSv3 client,
    for NFSv3 DSes).  From the destination DSes' perspective, the
    mover is an ordinary client issuing CHUNK_WRITE / CHUNK_FINALIZE /
    CHUNK_COMMIT.

4.  Reports progress and completion to the MDS, which then
    reconciles the two layouts and releases the proxy.

### DS Capability Registration

A DS advertises mover capability at EXCHANGE_ID time via a new flag
bit in `eia_flags`:

~~~
const EXCHGID4_FLAG_CAN_DATA_MOVE = 0x00200000;  // TBD -- aligns
                                                 // with existing
                                                 // EXCHGID flag space
~~~

Only DSes that set this flag MAY be selected as data movers.  A DS
that sets it MUST:

-  Support the data-mover callback ops defined below.
-  Be able to act as an NFSv4.2 client to peer DSes (establish
   sessions, issue CHUNK_WRITE/FINALIZE/COMMIT, renew leases).
-  Accept client I/O for proxied files on behalf of the layout's
   original mirror set, i.e., accept CHUNK ops against files the
   mover DS does not normally host.

The MDS tracks which DSes have registered mover capability and
selects among them when a transition is initiated.

### Mover Selection

Selection policy is implementation-defined.  The MDS MAY prefer:

-  A DS already in the source or destination mirror set (saves one
   hop on at least one side).
-  A DS colocated with the largest client population (reduces
   network distance for the proxy path).
-  A dedicated mover-only DS (isolates move load from normal
   serving load).

The selected DS MUST have `EXCHGID4_FLAG_CAN_DATA_MOVE` set.  If no
eligible DS is available, the MDS MUST NOT attempt the transition;
the source layout remains in effect and the MDS retries selection
later or escalates (for maintenance: block the DS offline; for
repair: return NFS4ERR_PAYLOAD_LOST via the existing per-chunk
escalation path).

## Proposal: Layout Shape During Transition

During PROXY_ACTIVE, the MDS issues layouts with three categories of
DS entry:

-  One or more **source-mirror DSes** holding the file's current data.
   Marked FFV2_DS_FLAGS_ACTIVE if readable, or FFV2_DS_FLAGS_REPAIR
   if degraded.  From the client's perspective these are *not*
   reachable during PROXY_ACTIVE.

-  One or more **destination-mirror DSes** being populated by the
   mover.  Marked with a new flag:

~~~
const FFV2_DS_FLAGS_MOVING_TO = 0x00000020;  // TBD
~~~

   These DSes accept writes from the mover (which is acting as a
   client to them) but not directly from the file's pNFS clients.

-  Exactly one **proxy DS entry** representing the mover.  Marked:

~~~
const FFV2_DS_FLAGS_PROXY = 0x00000040;  // TBD
~~~

When the PROXY flag is set on any data server entry in a layout,
clients MUST direct all I/O for this file to that DS rather than to
the mirror-set DSes.  The proxy DS internally dispatches reads and
writes to the source and destination mirrors as appropriate.

Open question: whether to use a single layout with the PROXY entry
or two distinct layouts (a "source" layout and a "destination"
layout) linked by a redirector record.  The one-layout approach is
simpler on the wire; the two-layout approach cleanly separates the
"what the client sees" layout from the "what the mover is building"
layout.  See Open Questions.

## Proposal: CB_PROXY Callback

When the MDS decides to place a file under proxy, it sends CB_PROXY
to each client that currently holds a layout for the file.  CB_PROXY
redirects client I/O to the mover.

~~~
/// struct CB_PROXY4args {
///     nfs_fh4              cpa_fh;              // file being proxied
///     stateid4             cpa_layout_stateid;  // client's current layout
///     ffv2_data_server4    cpa_proxy_ds;        // mover DS address
///     uint32_t             cpa_flags;           // see below
/// };
///
/// union CB_PROXY4res switch (nfsstat4 cpr_status) {
///     case NFS4_OK:
///         void;
///     default:
///         void;
/// };
~~~

### cpa_flags

~~~
const CB_PROXY_FLAG_MOVE            = 0x00000001;
const CB_PROXY_FLAG_REPAIR          = 0x00000002;
const CB_PROXY_FLAG_MAINTENANCE     = 0x00000004;
const CB_PROXY_FLAG_POLICY          = 0x00000008;
const CB_PROXY_FLAG_TLS_TRANSITION  = 0x00000010;
const CB_PROXY_FLAG_FH_TRANSITION   = 0x00000020;
~~~

The flags are informational to the client (diagnostics, logging,
retry policy) and carry no normative effect.  The MDS SHOULD set the
flag that best describes the transition.

### Client Behavior on CB_PROXY

After the client acknowledges CB_PROXY with NFS4_OK, it MUST:

1.  Stop issuing I/O directly to the source-mirror DSes for
    cpa_fh.
2.  Start routing all I/O for cpa_fh through cpa_proxy_ds.
3.  Continue to use its existing layout stateid; the proxy DS
    accepts CHUNK ops under that stateid.
4.  NOT issue LAYOUTGET for this file while proxy mode is active.
    (A subsequent LAYOUTGET is only valid after the MDS issues
    CB_LAYOUTRECALL -- see State Machine below.)

In-flight CHUNK ops that were already sent to source-mirror DSes
before the client processes CB_PROXY MAY complete at those DSes;
their results remain valid (they were written before the proxy
redirect).  New ops issued after the client has acked CB_PROXY MUST
go through the proxy DS.

A client MAY return NFS4ERR_NOTSUPP to CB_PROXY.  In that case, the
MDS MUST recall the client's layout (CB_LAYOUTRECALL) and either
serve the client via InBand I/O through itself during the transition
or refuse to reissue a layout for the file until the transition
completes.

### Late-Arriving Clients

Clients that call LAYOUTGET on a proxied file receive a layout in
which the PROXY flag is already set on the proxy DS entry.  They
proceed in proxy mode from the start, without ever receiving
CB_PROXY (there was nothing to redirect).

## State Machine

~~~
                  admin / policy / repair / maintenance
                                 |
                                 v
                          +---------------+
                          |   PRE_MOVE    |
                          | source layout |
                          |    only       |
                          +-------+-------+
                                  |
                                  | MDS selects mover,
                                  | creates destination
                                  | layout shape,
                                  | sends CB_PROXY to all
                                  | current clients
                                  v
                          +---------------+
                          | PROXY_ACTIVE  |
                          | mover handles |
                          | all client I/O|
                          | mover pushes  |
                          | to dest DSes  |
                          | in parallel   |
                          +-------+-------+
                                  |
                                  | mover reports completion
                                  | (all dest DSes have
                                  | COMMITTED content)
                                  v
                          +---------------+
                          |   COMMITTING  |
                          | MDS issues    |
                          |CB_LAYOUTRECALL|
                          | for the old   |
                          | proxied layout|
                          +-------+-------+
                                  |
                                  | all clients have
                                  | returned the proxied
                                  | layout
                                  v
                          +---------------+
                          |   POST_MOVE   |
                          | new layout    |
                          | active,       |
                          | source DSes   |
                          | released or   |
                          | marked for    |
                          | cleanup       |
                          +---------------+
~~~

### Transitions

| From | To | Trigger | Actions |
|------|-----|---------|---------|
| PRE_MOVE | PROXY_ACTIVE | admin / policy / repair / maintenance | MDS selects mover, sets PROXY flag in layouts, fans out CB_PROXY |
| PROXY_ACTIVE | COMMITTING | mover reports completion | MDS begins CB_LAYOUTRECALL fan-out |
| COMMITTING | POST_MOVE | all clients have LAYOUTRETURNed | MDS marks source DSes for cleanup, reissues normal layouts |
| PROXY_ACTIVE | PRE_MOVE | mover failure + no replacement | MDS cancels transition, unsets PROXY flag, CB_PROXY(proxy_ds=NULL) or equivalent to clear proxy |

## Client Behavior During Proxy Mode

1.  **CHUNK_READ / READ**: the client issues the read to the proxy
    DS using the existing layout stateid.  The proxy DS reads from
    the source mirrors (or reconstructs from surviving shards, in
    the repair case) and returns data to the client.

2.  **CHUNK_WRITE / WRITE**: the client issues the write to the
    proxy DS.  The proxy DS writes to the destination mirrors and
    MAY also write to the source mirrors to keep them current until
    the move is committed; whether to dual-write is an
    implementation choice balancing move-abort recovery cost
    against mover throughput.

3.  **CHUNK_FINALIZE / CHUNK_COMMIT**: same pattern -- the proxy DS
    is the single destination from the client's perspective.

4.  **chunk_guard4 CAS**: the proxy DS enforces the normal guard
    CAS on the destination mirror set.  Concurrent client writes
    during proxy mode serialize exactly as they do in normal
    operation, except that the serialization point is the proxy
    DS.  The proxy DS MAY translate guard values between the source
    and destination namespaces (e.g., if the source used a different
    coding type); the details are out of scope for this design and
    are internal to the mover.

5.  **Client stateids and FHs**: unchanged.  The client's layout
    stateid remains valid throughout proxy mode.  The client's
    filehandle does not change (it is an MDS filehandle, not a DS
    filehandle).

6.  **LAYOUTERROR**: if the client observes errors from the proxy
    DS (NFS4ERR_DELAY, connection failure, NFS4ERR_BAD_STATEID), it
    reports via LAYOUTERROR as usual.  The MDS interprets a
    proxy-DS LAYOUTERROR as a signal to consider mover replacement
    (see Mover Failure below).

## Mover Failure

If the mover DS crashes or becomes unreachable during PROXY_ACTIVE:

1.  Client I/O to the proxy DS receives NFS4ERR_DELAY (if the DS is
    reachable but unhealthy) or connection errors (if unreachable).
    Clients report LAYOUTERROR to the MDS.

2.  The MDS MAY select a replacement mover and re-issue CB_PROXY
    pointing at the replacement.  The new mover inherits the
    partial progress of the failed mover by:

    -  Reading whatever chunks the old mover managed to push to
       destination DSes (destination DSes persist chunks via the
       normal PENDING / FINALIZED / COMMITTED state machine; a
       half-done mover leaves partial COMMITTED state on the
       destinations).
    -  Re-reading from source DSes for chunks that were not yet
       pushed.

3.  If the MDS cannot find a replacement mover within a policy
    timeout, it MUST cancel the transition: send CB_LAYOUTRECALL,
    revert to the source layout (without the PROXY flag), and mark
    the destination DSes as FFV2_DS_FLAGS_REPAIR for a future
    attempt.

4.  Cascading mover failure (replacement mover also crashes)
    SHOULD trigger escalation to the admin rather than recursive
    retry.  Recurring failures likely indicate an environmental
    issue the mover cannot work around.

## Whole-File Repair as a Specialization

Whole-file repair is a Data Mover operation where the source mirror
is degraded.  The mover reads from whatever shards remain with
integrity, reconstructs missing data via the erasure code (when
possible), and writes to the destination layout.  If surviving
shards are insufficient (fewer than k shards with matching guards
across the mirror set), the mover MUST terminate the transition
with NFS4ERR_PAYLOAD_LOST for the affected byte ranges, following
the per-chunk repair semantics in sec-repair-selection of the main
draft.

The difference from per-chunk (CB_CHUNK_REPAIR) repair:

-  CB_CHUNK_REPAIR delegates per-range work to a pNFS CLIENT that
   the MDS selects.  The client must already hold a layout for the
   file and is limited to the ranges the MDS passes in.

-  Whole-file repair uses a DATA MOVER (a DS), which does not
   require an existing client and can operate on files with no
   active opens.  The mover addresses the whole file in one
   operation rather than many small CB_CHUNK_REPAIR calls.

When to use which:

| Scenario | Prefer |
|----------|--------|
| A few chunks damaged, active clients present | CB_CHUNK_REPAIR |
| Many chunks damaged | Data Mover whole-file repair |
| No active clients | Data Mover whole-file repair |
| File being simultaneously moved for policy | Data Mover (combine move + repair) |

Both paths use the same correctness model (lowest-guard-recoverable,
payload consistency and integrity checks) and both terminate with
NFS4ERR_PAYLOAD_LOST on permanent loss.

## DS Crash Recovery

Destination DSes are normal DSes: they participate in the existing
DS crash recovery story (sec-tight-coupling-ds-crash in the main
draft).  If a destination DS crashes during PROXY_ACTIVE:

-  The mover handles the failure the same way a client would:
   LAYOUTERROR to the MDS, which MAY substitute a spare
   (FFV2_DS_FLAGS_SPARE in the layout) or mark the destination as
   FFV2_DS_FLAGS_REPAIR.

-  The mover continues pushing to the remaining destinations.
   Clients continue to see the file via the proxy DS; their I/O
   is unaffected.

-  If the number of healthy destination DSes falls below the
   erasure code's minimum (k of k+m), the MDS MUST either
   substitute from a spare pool or abort the transition.

Source DS crashes during PROXY_ACTIVE reduce the mover's read
parallelism but do not block forward progress as long as the erasure
code can still reconstruct from surviving source DSes.  If the
source degrades past reconstructibility, the move transitions to
whole-file repair semantics automatically: partial reconstruction
succeeds; ranges that cannot be reconstructed terminate with
NFS4ERR_PAYLOAD_LOST.

## MDS Crash Recovery

If the MDS crashes during PROXY_ACTIVE:

1.  Clients and movers detect the MDS session loss and enter
    RECLAIM per {{RFC8881}}.

2.  On reconnection, the MDS re-advertises the proxy layout (with
    PROXY flag still set) to each reclaiming client.

3.  The mover, also a client to the destination DSes, reclaims its
    own layouts on those destinations.

4.  If the MDS persisted its view of the move (which
    destination DSes are populated, which chunks are committed,
    etc.), the mover resumes from the last persisted checkpoint.
    Persistence of move state is RECOMMENDED but not required; a
    non-persistent MDS implementation restarts the move from the
    beginning on crash.

5.  If the MDS crashed while the transition was between
    PROXY_ACTIVE and COMMITTING (i.e., the mover had reported
    completion but the MDS had not yet issued CB_LAYOUTRECALL),
    the MDS on restart SHOULD re-drive the COMMITTING phase.

## Backward Compatibility

### Clients Without CB_PROXY Support

Clients that do not support CB_PROXY MUST NOT be given a layout for
a file that is in PROXY_ACTIVE.  The MDS options are:

1.  Refuse LAYOUTGET with NFS4ERR_LAYOUTUNAVAILABLE.  The client
    falls back to InBand I/O through the MDS.

2.  If the MDS is willing to proxy I/O itself (taking the mover's
    role), it can serve the client's reads and writes through its
    own data path while the actual move happens in parallel.  This
    is operationally expensive and is not recommended for large
    transitions.

Clients that advertise CB_PROXY support via a capability bit on
EXCHANGE_ID (TBD: BACKCHANNEL_SUPPORT equivalent) are assumed to
accept CB_PROXY; the MDS does not test each file individually.

### DSes Without Mover Capability

DSes that do not set `EXCHGID4_FLAG_CAN_DATA_MOVE` MUST NOT be
chosen as movers.  A deployment that has no mover-capable DSes
cannot perform transitions; it falls back to:

-  Per-chunk CB_CHUNK_REPAIR for single-shard repair.
-  Admin-coordinated offline procedures for policy transitions.
-  Blocking DS maintenance (the DS cannot drain its files).

Deployments SHOULD ensure at least one mover-capable DS exists per
failure domain to avoid single-point-of-failure on move operations.

### NFSv3 Source DSes

When the source mirror is an NFSv3 DS (Flex Files v1 compatibility),
the mover reads from it using NFSv3 semantics.  The mover converts
to NFSv4.2 CHUNK semantics for the destination if the destination
is an NFSv4.2 DS.  This is the same pattern the main draft uses
for InBand I/O; see dstore-vtable-v2.md.

## Security Considerations

1.  **Mover authority.** The mover DS receives all client I/O for
    the proxied file.  A compromised mover can observe or modify
    file data.  Deployments MUST treat mover-capable DSes as at
    least as trusted as the DSes they can proxy for.  The
    `EXCHGID4_FLAG_CAN_DATA_MOVE` bit SHOULD be gated by a
    deployment-level allowlist or an MDS-operator policy.

2.  **Transport security across the move.** The mover's connections
    to source and destination DSes are independent of the client's
    connection to the mover.  A mover MAY read from an AUTH_SYS
    source and write to a TLS destination (or vice versa).  The
    mover is responsible for enforcing the effective security
    policy (e.g., do not downgrade encrypted data to a plaintext
    DS).

3.  **Principal binding during proxy mode.** When tight coupling
    (see sec-tight-coupling-control in the main draft) is in use,
    the mover DS MUST present a principal to source and destination
    DSes that those DSes will accept.  The client's authenticated
    identity is NOT automatically forwarded through the mover;
    unless the mover re-authenticates as the client (which
    typically requires constrained delegation or equivalent), the
    mover's access to peer DSes is under the mover's own service
    identity.  See Open Questions.

4.  **Mover impersonation.** A malicious MDS could send CB_PROXY
    to redirect a client to a hostile mover.  The existing MDS
    trust model already grants the MDS this capability via
    CB_LAYOUTRECALL + new layout; CB_PROXY does not weaken it.
    Clients that require stronger mover identity verification
    SHOULD validate the mover's transport-security credentials
    against a deployment allowlist.

## Interaction with Existing Protocol Elements

### chunk_guard4

The proxy DS enforces chunk_guard4 CAS on the destination mirror
set on behalf of clients.  In the common case (single transaction
at a time), the guard values the client sends through the proxy are
translated by the mover to guard values appropriate for the
destination DSes; the mover MAY use the same guard values or
generate fresh ones, provided uniqueness on the destination side
is preserved.

### CHUNK_LOCK

If a client holds a chunk lock on a file when the file enters
PROXY_ACTIVE, the lock follows the file: the mover takes ownership
of the lock on behalf of the original holder, and the MDS-escrow
semantics (sec-chunk_guard_mds in the main draft) apply if the
original holder becomes unreachable during the move.

### CB_CHUNK_REPAIR

CB_CHUNK_REPAIR (per-chunk repair) and CB_PROXY (mover-driven move)
can coexist on the same file at different times but SHOULD NOT be
active simultaneously.  The MDS SHOULD not issue CB_CHUNK_REPAIR on
a proxied file; the mover handles any mid-move repair internally.
If the MDS decides a proxied file also needs per-chunk repair, it
SHOULD cancel the proxy, perform the per-chunk repair via
CB_CHUNK_REPAIR, then re-initiate the proxy.

## Open Questions

1.  **One layout or two?** The design above uses a single layout
    with a PROXY-flagged DS entry.  An alternative is to issue two
    separate layouts (one for the source, one for the destination)
    with a redirector record pointing at the mover.  The one-
    layout approach is simpler on the wire but conflates two
    conceptually distinct views.  WG input needed.

2.  **TLS transition wire shape.** This design folds TLS
    transitions into the general CB_PROXY mechanism with a
    TLS_TRANSITION flag.  Is a distinct callback warranted, or is
    the flag sufficient?  Are there TLS-specific normative rules
    the MDS must enforce (e.g., the destination DS MUST have at
    least as strong a security profile as the source)?

3.  **Filehandle transition detail.** The use case is real but the
    mechanism specified here assumes the mover reconciles old-to-
    new FHs internally.  Do clients need any notification beyond
    CB_PROXY?  Does the NFS filehandle they hold actually change
    when the file moves?  (The MDS filehandle does not change; the
    DS filehandles change but are opaque to the client in layout
    form.)

4.  **Per-range scope.** Should CB_PROXY be per-range (move half
    the file at a time) or always whole-file?  Per-range enables
    moves of very large files without funneling the entire file
    through one mover.  Adds complexity in the state machine.

5.  **Transitive proxy.** If a file in PROXY_ACTIVE needs a second
    move (e.g., DS maintenance interrupts a repair), what
    happens?  Queue the second move?  Abort the first?
    Transitive proxy (mover-of-mover)?

6.  **Delegation-based principal forwarding.** For tight-coupling
    deployments, how does the mover present itself to destination
    DSes?  Options: (a) always as its own service identity,
    (b) with constrained delegation from each client, (c) with a
    per-move MDS-minted principal that destination DSes accept.
    Each has tradeoffs in security model and deployment
    complexity.

7.  **MDS persistence of move state.** Is it a SHOULD or MAY?  The
    tradeoff is restart cost against per-move persistence overhead.
    Likely SHOULD for production and MAY for prototype.

8.  **Lustre interop benchmark.** Is there a concrete performance
    target (e.g., "whole-file repair of a 1 GB file must complete
    in under T seconds at the cluster's current available
    bandwidth")?  Without a target, "beat Lustre" is aspirational
    rather than testable.

## Deferred / NOT_NOW_BROWN_COW

-  Partial-file moves (moving a byte range while the rest of the
   file stays on the source).
-  Multi-mover pipelines for very large files.
-  Automated mover selection with load balancing across
   mover-capable DSes.
-  Mover failure predicate (when should the MDS pre-emptively
   replace a slow mover?).
-  Integration with server-side copy ({{RFC7862}} Section 4) as
   an alternative for single-file moves within one namespace.

## Relation to stateids.md

The trust-stateid design in stateids.md defines the MDS-to-DS
control-plane primitive for stateid registration and revocation.
This design defines the MDS-to-DS coordination primitive for layout
transitions.  Both are MDS control-plane extensions; neither
requires the other but they compose naturally:

-  A proxied file under tight coupling has a trust entry on the
   mover DS for the client's layout stateid.  The MDS issues
   TRUST_STATEID on the mover when the proxy is established, just
   as it would for any other DS in a tight-coupling deployment.
-  On proxy completion, REVOKE_STATEID on the retired source
   DSes cleans up their trust entries.

## Key Files (if folded into the main draft)

| File | Change |
|------|--------|
| `draft-haynes-nfsv4-flexfiles-v2.md` | Add `# Data Mover` section; rewrite §10.6 as pointer; add CB_PROXY XDR + description in `# New NFSv4.2 Callback Operations` |
| `lib/xdr/nfsv42_xdr.x` | Add CB_PROXY op, EXCHGID4_FLAG_CAN_DATA_MOVE, FFV2_DS_FLAGS_MOVING_TO, FFV2_DS_FLAGS_PROXY |
| Future implementation in reffs | New `data_mover` subsystem in lib/nfs4/server/; new `mover_session` abstraction for mover-as-DS-client pattern |
