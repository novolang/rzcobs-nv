# rzcobs-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Reverse zero-compressing COBS: the framing `defmt` sends its logs in.
Like COBS it removes every zero byte from a payload so that a zero can
end a frame.  Unlike COBS it spends the header byte it needed anyway on
a **bitmap of which bytes were zero**, so a payload full of zeros comes
out shorter than it went in — and it puts each header **after** the
bytes it covers, so the encoder never seeks backwards.

A log frame is mostly zeros: an interned format index whose value is 12
in a 32-bit field, an argument that is 0, a timestamp whose top bytes
have not moved since the last line.  That is why `defmt` chose this over
COBS, and it is why the device side of novo-lang's deferred logging
needs it.

## Adding it, and checking it

```bash
novo pkg add rzcobs-nv       # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/rzcobs_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first three constants fails with `not implemented:
rzcobs.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use rzcobs

// Thirteen zeros and a byte — fourteen bytes of payload — leave in a
// four-byte frame, terminator included.
fn main() [io]
    let payload = bytes.from_hex("00000000000000000000000000ff") ?? bytes.zeros(0)
    println(bytes.to_hex(rzcobs.encode(payload)))   // 7fff3f00
```

## How it differs from cobs-nv, and when each is right

Both packages remove zeros so that a zero can delimit a frame.  Three
things separate them, and each one decides a case.

| | cobs-nv | rzcobs-nv |
| --- | --- | --- |
| where the header sits | before the block it covers | **after** the chunk it covers |
| a payload of zeros | one byte of overhead per 254, always | one byte **total** per seven zeros |
| decoding | forward, streaming, one pass | **backwards**, whole frame first |
| the terminator | the caller's to write | this package writes it — 0x00 is a value in this format's own alphabet |
| overhead bound | `n / 254 + 1` | `ceil(n / 134) + 1` |
| round trip | exact | the payload plus **up to six zero bytes** |

**Reach for cobs-nv** when the receiver decodes as bytes arrive, when
the payload has to come back byte-exact, or when the other end is
already speaking COBS — a UART link, `postcard-rpc`, an existing
protocol.

**Reach for rzcobs-nv** when the sender is a device with an interrupt to
get out of and the payload is structured binary with zeros in it — a log
frame, a telemetry record — and when the receiver is a host that has the
whole frame anyway.  The reverse header is not a curiosity: it means the
encoder is append-only, so it can write straight into a queue that only
moves forward, with no reserved byte to come back and patch.  cobs-nv's
own `encode_into` has a `patch()` that seeks; this one has nothing to
seek to.

The two costs are real and are stated rather than buried.  Decoding
needs the whole frame, so there is no streaming decoder here and there
will not be one.  And the round trip appends up to six zeros, because
the last chunk is padded to seven slots and a decoder cannot tell a sent
zero from a padding one; `defmt` lives with this because its frames are
self-delimiting a layer up, and a caller who cannot must carry its own
length.

## The layer, and why

`core`.  Everything here is arithmetic over bytes the caller already
holds — nothing is read, nothing is written, and the encoder's state is
a value the caller owns rather than a buffer the package hides.  No
function declares an effect at all, because a framing has nowhere to put
one.

The package is **two modules on purpose**, and the split is the device
claim:

* `rzcobs_core` is the encoder as a `@value` state machine over
  integers.  It builds for a Cortex-M, and `tests/embedded_probe.nv` is
  that claim in a form the shard audit either links or does not.
* `rzcobs` is the same format over `Bytes` and `Cursor`, plus the
  decoder.  It does **not** build for a device, and it is not supposed
  to: the embedded runtime defines no `novo_bytes_*` symbol, and one
  host-only function anywhere in a compilation unit is an undefined
  symbol at link time whether or not the firmware calls it.  So the
  probe uses `rzcobs_core` and nothing else, and the audit's module walk
  leaves `rzcobs` out by following the probe's own `use` lines.

## The load-bearing interface

Two decisions, and the second follows from the first.

**The header follows its chunk.**  That is the whole of "reverse", and
it is what makes the encoder a forward-only state machine:

```novo
pub @value
struct RzEncoder
    run: Int
    zeros: Int

pub @value
struct RzEmit
    enc: RzEncoder
    out: [u8; 2]
    len: Int

pub fn push(e: RzEncoder, b: u8) -> RzEmit
pub fn finish(e: RzEncoder) -> RzEmit
```

One byte in, at most two bytes out — a literal, and the header that
closes the chunk it filled.  `out` is a fixed two-byte inline array
rather than a list because a list is a heap allocation and this runs in
an interrupt; two is the bound, and it is asserted in the test file
rather than promised in a comment.

**The state is a `@value` struct, so it cannot report through a
`Result`.**  A `@value` struct is unboxed, and SPEC § 14.5 excludes it
from `Result` payloads, optional payloads and fields of boxed structs.
An encoder that answered `Result<RzEncoder, _>` would not compile, and
one that boxed itself would allocate on a device.  What is left — and
what is published — is a total function returning one unboxed value that
carries both the new state and the bytes.  The encoder has no failure
mode, so this costs nothing here; `bbqueue-nv`'s README carries the same
constraint where it does cost something.

The decoder's counterpart is `decoded_len`, and it is public because the
format made it necessary:

```novo
pub fn decoded_len(wire: Bytes) -> Result<Int, RzcobsError>
```

A reverse decoder reads the LAST chunk first, so it cannot place its
first output byte until it knows where the output ends.  Rather than
hide a two-pass walk inside `decode_into` and leave a caller sizing a
buffer to guess, the first pass is a function anyone can call.

## The alphabet, exactly

Each header byte covers the literal bytes that **precede** it.

| header | meaning | output bytes covered |
| --- | --- | --- |
| `0x00` | end of frame | — |
| `0x01`–`0x7F` | a seven-slot bitmap.  Bit *i*, LSB first: 1 means the *i*th byte was `0x00` and is not on the wire; 0 means take one literal from the stream. | exactly 7 |
| `0x80`–`0xFE` | `1nnnnnnn`: take *n* + 7 literals, then output one `0x00`. | 8–134 |
| `0xFF` | take 134 literals, and output **no** trailing zero. | 134 |

A bitmap of zero would be the byte `0x00`, which ends the frame — so
seven non-zero bytes cannot be a bitmap.  They open a run instead, which
is exactly why the `0x80` family exists.  Follow that one rule and every
vector in `tests/rzcobs_tests.nv` falls out of it.

## The reference implementation

The `rzcobs` Rust crate (MIT/Apache-2.0), which is `defmt`'s default
framing, and the format description in the defmt book's encoding
chapter.  Every vector in `tests/rzcobs_tests.nv` is the crate's own,
with one difference the test file names: the crate's `encode` answers
the frame and leaves the terminator to its caller, and this package
writes the terminator, so each expected encoding here carries a trailing
`00` the crate's does not.  The payloads and the header bytes are
unchanged, which is what wire compatibility means — `defmt-print` and
`probe-rs run` decode what this produces.

## Status

| item | implemented |
| --- | --- |
| `rzcobs_core.bitmap_span`, `.run_span`, `.terminator` | no |
| `rzcobs_core.encoder`, `.push`, `.finish` | no |
| `rzcobs_core.max_encoded_len` | no |
| `rzcobs.max_encoded_len` | no |
| `rzcobs.encode_into`, `.encode` | no |
| `rzcobs.decoded_len`, `.decode_into`, `.decode` | no |
| `rzcobs.frame_end` | no |
| `rzcobs.RzcobsError.message` | no |
