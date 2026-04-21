# flexfiles-v2-data-mover

Work-in-progress companion draft to
[`draft-haynes-nfsv4-flexfiles-v2`](https://github.com/ietf-wg-nfsv4/flexfiles-v2)
covering the Data Mover mechanism.

## Status

**Design phase.** No Internet-Draft has been submitted yet. The current
artifact is [`design.md`](design.md), a design document in the same
style as
[`stateids.md`](https://github.com/ietf-wg-nfsv4/flexfiles-v2/blob/main/stateids.md)
in the main repository.

A first-pass technical review has been done; the findings (three
BLOCKERs and a set of WARNINGs) are filed as issues in this repository
under the `blocker`, `warning`, and `open-question` labels. They must
be resolved before the first `-00` draft.

## Scope

The mechanism addresses six use cases that currently fall outside
`draft-haynes-nfsv4-flexfiles-v2`:

1. Administrative ingest (rsync into cluster, layout policy applied)
2. Policy-driven layout transition (e.g., mirror → erasure coded)
3. Data server maintenance / evacuation
4. Whole-file repair (the open §10.6 cref in the main draft)
5. TLS coverage transition
6. Filehandle / storage-backend transition

All six converge on a single primitive: a designated data server
("mover") becomes the authoritative source of truth for client I/O
on a file during a transition, pushing data to the destination layout
in the background. A new CB_PROXY callback redirects clients to the
mover.

## Relation to the main draft

- `draft-haynes-nfsv4-flexfiles-v2` §10.6 (Whole File Repair) will
  contain a short pointer to this document when this work reaches
  `-00`. Until then, the main draft's §10.6 is a WIP reference to
  this repository.
- This work depends on and composes with the main draft's
  `chunk_guard4` (sec-chunk_guard4), `CB_CHUNK_REPAIR`
  (sec-CB_CHUNK_REPAIR), and tight-coupling control protocol
  (sec-tight-coupling-control). Pre-existing protocol machinery is
  reused; this design does not duplicate it.
- The companion design document `stateids.md` in the main repository
  covers the MDS-to-DS trust-stateid control plane; this design
  covers the MDS-to-DS and MDS-to-client coordination for layout
  transitions. They are orthogonal.

## Open questions to resolve before -00

- Principal forwarding across the mover boundary (OQ6 in
  `design.md`).
- One layout with PROXY flag vs two layouts with a redirector (OQ1).
- Transitive proxy semantics when a second transition is triggered
  during an active one (OQ5).
- Cancel-path XDR gap: `CB_PROXY_CANCEL` vs a cancel flag on
  `CB_PROXY`.
- `chunk_guard4` translation semantics at the proxy boundary —
  correctness rule for preserving per-chunk linearizability when
  the mover serializes writes on behalf of clients.
- COMMITTING-phase mover failure recovery — state machine currently
  has no exit transition.

## License

AGPL-3.0-or-later is inherited from the main draft's repository
practice; prose contributions are under the IETF trust licensing
terms once an Internet-Draft is submitted.

## Contact

loghyr@gmail.com. Discussion should happen on the NFSv4 WG list
(nfsv4@ietf.org) once the work is submitted as a draft; until then,
use this repository's issues.
