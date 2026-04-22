# flexfiles-v2-data-mover

Work-in-progress companion Internet-Draft to
[`draft-haynes-nfsv4-flexfiles-v2`](https://github.com/ietf-wg-nfsv4/flexfiles-v2)
covering the Data Mover mechanism.

## Status

**Pre-submission.** The draft source is
[`draft-haynes-nfsv4-flexfiles-v2-data-mover.md`](draft-haynes-nfsv4-flexfiles-v2-data-mover.md).
It has not yet been submitted to the datatracker. Ongoing design
iteration happens in this repository; issues in the flexfiles-v2
repository's `data-mover` label track the findings being addressed.

The build uses Martin Thomson's i-d-template: run `make` to produce
`.txt` and `.html`; `make idnits` to run `idnits` against the rendered
output.

## Scope

The mechanism addresses three classes of work that fall outside the
per-chunk repair in
[`draft-haynes-nfsv4-flexfiles-v2`](https://github.com/ietf-wg-nfsv4/flexfiles-v2):

1. **Whole-file repair** when per-chunk reconstruction is not viable.
2. **Layout transitions** for policy, maintenance, or environmental
   reasons (coding-type migration, DS evacuation, TLS coverage
   transition, filehandle migration).
3. **Codec translation** for clients that cannot participate in the
   file's native erasure code, including NFSv3 clients.

The mechanism is a **registered proxy** that receives MDS-initiated
directives on a dedicated control session and provides the
server-side of a client-facing data path during the operation.

## Relation to the main draft

- `draft-haynes-nfsv4-flexfiles-v2` and this draft are mutually
  referenced; the main draft's repair-selection section points here
  for the whole-file case, and this draft cites the main draft for
  all of its base machinery (chunk_guard4, tight-coupling control
  protocol, CB_CHUNK_REPAIR).
- This draft normatively depends on
  `draft-haynes-nfsv4-flexfiles-v2`; per-chunk and whole-file
  operations are mutually exclusive for a given file at a given time.

## License

AGPL-3.0-or-later inherits from the main repository's practice;
prose contributions are under the IETF trust licensing terms once
an Internet-Draft is submitted.

## Contact

loghyr@gmail.com. Protocol discussion happens on the NFSv4 WG list
(nfsv4@ietf.org) once the work is submitted; until then, open an
issue in this repository.
