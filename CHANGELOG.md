# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
