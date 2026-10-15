# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-10-10

The **implementation**, 0.0.2's interface with the bodies behind it.  No
signature, type or effect row changed.

- `rzcobs_core.push` and `.finish` are the state machine the module
  header describes: a chunk below seven bytes accumulates in the
  bitmap, the seventh closes it unless the bitmap came out empty — seven
  non-zero bytes open a run instead — and a run closes at 134 with
  `0xFF`.  `finish` pads a short chunk with `((1 << (7 - run)) - 1) <<
  run`.  Every step answers at most two bytes and nothing allocates.
- `rzcobs` reuses the core's constants instead of restating them, so
  the two halves agree on the format by construction.  The test that
  compares `max_encoded_len` across the halves is structural rather
  than coincidental.
- A decoder names the earliest fault rather than the first one a
  backward walk meets.  A wire that does not end in `0x00` is
  `RzcobsUnterminated`; one holding a `0x00` before the end is
  `RzcobsZeroInFrame` at its offset; only a wire holding neither
  reaches the header walk, which answers `RzcobsTruncated` at the
  header covering more literals than precede it.  The zero scan has to
  run before the walk: walking back from the terminator of `01000000`
  would name the wrong fault.
- The backward walk establishes the output length, and placement seeks
  each chunk to the offset its header decides, so `decode_into`
  allocates nothing and `decoded_len` is public for a caller sizing the
  destination the same way.
- `tests/rzcobs_edge_tests.nv` covers what the crate's vectors do not:
  frames built from `0x80`, `0x81`, `0xFE` and `0xFF` run headers, an
  empty frame, and the text of each refusal.

The two caveats from 0.0.1 stand unchanged.  The round trip is still
**not exact**, and there is still **no streaming decoder**.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-10

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- `RzEncoder` and `RzEmit` — the encoder as a `@value` state machine
  over two integers, and one byte in / at most two bytes out.  `out` is
  a fixed two-byte inline array because two is the bound and a list
  would be a heap allocation in the interrupt this runs in.
- `rzcobs_core.push` and `.finish`, which are the whole format: a chunk
  of seven bytes becomes a bitmap header, seven non-zero bytes open a
  run instead because a bitmap of zero is the terminator, and a run
  closes at 134 with `0xFF`.
- `rzcobs.encode_into` / `.encode` and `.decode_into` / `.decode` over
  `Bytes` and `Cursor`, with `RzcobsError` naming the offset a frame
  stopped making sense at.
- `rzcobs.decoded_len`, public because the decode runs backwards: the
  last chunk is read first, so neither the decoder nor a caller sizing a
  buffer can know where the output ends without one walk of the headers.
- `rzcobs.frame_end`, so a reader splits a stream on the one byte an
  encoded frame cannot contain.

Two facts a consumer has to know before depending on this, both stated
in the README rather than discovered later.  The round trip is **not
exact**: a message comes back with up to six zero bytes appended,
because the last chunk is padded to seven slots.  And there is **no
streaming decoder**, here or ever: a header follows the bytes it covers,
so decoding starts at the end of a frame and needs all of it.
