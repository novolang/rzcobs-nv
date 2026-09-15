# rzcobs-nv

Reverse zero-compressing COBS is a framing: it removes every zero byte
from a payload, so that a zero byte can mark the end of a frame. It is
the framing the Rust logging library
[defmt](https://defmt.ferrous-systems.com/encoding) sends its log frames
in, and the
[rzcobs crate](https://docs.rs/rzcobs) is its reference implementation.
This package brings the format to novo-lang: an encoder a device can run
in an interrupt handler, and a decoder for the host that reads the logs.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the format is

A framing turns a stream of bytes into a sequence of messages. The usual
way is to pick one byte value as the end-of-frame marker and remove that
value from the payload. Consistent Overhead Byte Stuffing (COBS) is the
classic scheme: it removes zero bytes by writing, before each block of
data, a **header** byte saying how far the next zero is.

rzCOBS changes two things about that. Its header comes **after** the
bytes it covers, and the header of a short chunk is a **bitmap** saying
which of those bytes were zero rather than a distance.

A bitmap header covers seven output bytes. Each of its seven low bits,
least significant first, says whether that byte was zero. A zero byte is
never written to the wire, so seven zeros cost one header byte and
nothing else. A log frame is mostly zeros, which is why defmt chose this
scheme: a 32-bit format index holding the value 12 is three zeros and a
byte.

A header that follows its chunk means the encoder never writes backwards.
It appends a literal or a header and moves on, so it can write straight
into a queue that only moves forward. The cost falls on the decoder: it
starts at the byte before the terminator and walks backwards, so it needs
the whole frame before it can begin.

These are the four header values, and everything else follows from them.

| Header | Meaning | Output bytes covered |
| --- | --- | --- |
| `0x00` | End of frame. | — |
| `0x01`–`0x7F` | A seven-slot bitmap. Bit *i*, least significant first: 1 means the *i*th byte was zero and is not on the wire; 0 means take one literal. | 7 |
| `0x80`–`0xFE` | `1nnnnnnn`: take *n* + 7 literals, then output one zero byte. | 8 to 134 |
| `0xFF` | Take 134 literals, and output no trailing zero. | 134 |

A bitmap of zero would be the byte `0x00`, which ends the frame. Seven
non-zero bytes therefore cannot be a bitmap, and they open a run
instead. That is what the `0x80` family is for.

| Quantity | Value |
| --- | --- |
| Output bytes one bitmap header covers | 7 |
| Literals one run header covers | 8 to 134 |
| The terminator | `0x00` |
| Longest encoding of `n` bytes, terminator included | `n + ceil(n / 134) + 1` |
| Encoding of 134 non-zero bytes | 136 bytes |
| Encoding of an empty payload | 1 byte |
| Bytes one encoder step can produce | 0, 1 or 2 |
| Zero bytes a round trip may append | 0 to 6 |

## Install

```
novo pkg add rzcobs-nv
```

## Example

```novo
use std.bytes
use rzcobs

fn main() [io]
    // Fourteen bytes of payload: thirteen zeros and one 0xFF.
    let payload = bytes.from_hex("00000000000000000000000000ff") ?? bytes.zeros(0)

    // Four bytes on the wire, terminator included. Two bitmap headers
    // stand for the thirteen zeros, and the 0xFF is the only literal.
    println(bytes.to_hex(rzcobs.encode(payload)))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test past the three constants reaches a
`not implemented: rzcobs.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `rzcobs_core` | The encoder as a state machine over integers: the three format constants, one byte in and at most two out, the closing header, and the length bound. It holds no buffer and allocates nothing. |
| `rzcobs` | The same format over buffers: encode and decode a whole payload or write into a cursor the caller owns, the length of a frame's output, the three named refusals, and the search for the next terminator in a stream. |

## How to choose an entry point

**`rzcobs.encode` and `rzcobs.decode` take a whole payload.** Each
allocates its answer at exactly the length it needed.

**`rzcobs.encode_into` and `rzcobs.decode_into` write into a `Cursor`
you already own.** Size the destination with `rzcobs.max_encoded_len`
before encoding, and with `rzcobs.decoded_len` before decoding.

**`rzcobs_core.encoder` is the encoder for firmware.** Feed it one byte
at a time with `push`, write the bytes each step answers, and call
`finish` at the end of the message. Nothing is allocated, so this is the
form for an interrupt handler. See "Running on a microcontroller".

**`rzcobs.frame_end` is what a stream reader needs.** It finds the next
terminator, and answers nothing when the frame is still arriving.

## The rules a user needs

1. **The round trip is not exact.** A decoded message comes back
   followed by up to six zero bytes. The last chunk is padded to seven
   slots, and a decoder cannot tell a padding zero from a sent one.
   defmt lives with this because its frames carry their own length one
   layer up. A caller who cannot must carry its own length too.
2. **A payload whose length is a multiple of seven round-trips
   exactly.** That is the same rule, from the other side.
3. **This package writes the terminator; COBS implementations usually do
   not.** The `0x00` is a header value in this format's own alphabet,
   and a frame without it cannot be decoded, because the decoder has
   nowhere to start.
4. **There is no streaming decoder, and there will not be one.**
   Decoding runs backwards from the terminator, so the whole frame must
   be in one place first.
5. **A frame contains exactly one zero byte, and it is the last one.**
   A stream therefore splits on the terminator and nowhere else.
   `rzcobs.frame_end` answers `None` for "read more", which is not a
   fault.
6. **Size a destination before writing into it.**
   `rzcobs.max_encoded_len(n)` is `n + ceil(n / 134) + 1`.
   `rzcobs.decoded_len(frame)` walks the headers once and answers the
   output length. A cursor with less room than that is a panic, as an
   index out of range is: a buffer the caller sized wrong is a mistake
   in the program.
7. **Decoding answers a value on a bad frame, never a panic.**
   `RzcobsZeroInFrame(at)` is a zero byte before the end,
   `RzcobsTruncated(at)` is a header claiming more literals than
   precede it, and `RzcobsUnterminated(len)` is a frame that does not
   end in a zero byte. Each carries an offset into the frame.
8. **The encoder has no failure mode.** `rzcobs_core.push` and `finish`
   are total functions answering the new state and the bytes to write.
   A `@value` struct cannot be a `Result` payload (SPEC section 14.5),
   and the format gives the encoder nothing to refuse.
9. **One encoder step writes at most two bytes.** That is a literal, and
   the header that closed the chunk it filled. The emit buffer is a
   fixed inline array of two for that reason.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `rzcobs_core` and nothing else. It takes
and answers `Int` and `u8`, and it holds no buffer.

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds today:

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

The probe produces a Cortex-M4 executable that reads the three
constants, runs the length bound, and drives the encoder byte by byte to
the closing header. It builds and it is not run: every function it calls
is a `todo()` today.

**A device cannot use the `rzcobs` module.** That module speaks `Bytes`
and `Cursor`, and the embedded runtime defines neither. One host-only
function anywhere in a compilation unit is an undefined symbol at link
time on a device, whether or not the firmware calls it. That is why the
package is two modules, and the decoder is in the host half: a device
sending logs never decodes one.

## What is not included

- **A streaming decoder.** See rule 4.
- **A decoder in the device half.** Decoding needs the whole frame and a
  buffer to put the output in, which is the host's side of a log link.
- **The defmt log format itself.** This package is the framing only. The
  bytes inside a frame are an interned format index and its arguments,
  and reading those is
  [deflog-decoder](https://novo-lang.org/packages/deflog-decoder)'s job.
- **A transport.** Nothing here reads or writes. RTT, a UART and a file
  are all the caller's.
- **An exact round trip.** See rule 1.

## Related packages

- [cobs-nv](https://novo-lang.org/packages/cobs-nv) is plain Consistent
  Overhead Byte Stuffing. Its header precedes the block it covers, it
  decodes forward as bytes arrive, its round trip is exact, and its
  overhead is one byte per 254. Reach for it when the receiver decodes a
  stream as it arrives, when the payload must come back byte-exact, or
  when the other end already speaks COBS.
- [frame-nv](https://novo-lang.org/packages/frame-nv) is length-prefixed
  framing, for a link where the payload need not be scanned at all.
- [deflog-parser](https://novo-lang.org/packages/deflog-parser) and
  [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) are the
  deferred-logging packages that read what arrives inside these frames.
- [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv) is the
  single-producer queue a device writes encoded frames into.

## Tests

```bash
novo test tests/rzcobs_tests.nv       # 19 tests
```

Every vector is the `rzcobs` crate's own, with one difference the test
file names: that crate's `encode` leaves the terminator to its caller
and this package writes it, so each expected encoding here carries a
trailing `00` the crate's does not. The payloads and the header bytes
are unchanged, which is what wire compatibility means:
`defmt-print` and `probe-rs run` read what this produces.

The suite asserts the three constants, the length bound of one header
per 134 bytes, that a run of zeros costs one byte per seven, that a
short chunk is padded to seven slots, that seven bytes with a zero are a
bitmap and seven without one open a run, that a full run closes with
`0xFF` and no trailing zero, that the round trip appends up to six
zeros, that a payload whose length is a multiple of seven round-trips
exactly, that the output length is known before the output is, that both
buffer-writing calls allocate nothing, that each of the three refusals
names its offset, that a stream splits on the terminator and nowhere
else, and that the device encoder and the buffer encoder agree byte for
byte.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `rzcobs_core.bitmap_span`, `.run_span`, `.terminator` | no |
| `rzcobs_core.encoder`, `.push`, `.finish` | no |
| `rzcobs_core.max_encoded_len` | no |
| `rzcobs.max_encoded_len` | no |
| `rzcobs.encode_into`, `.encode` | no |
| `rzcobs.decoded_len`, `.decode_into`, `.decode` | no |
| `rzcobs.frame_end` | no |
| `rzcobs.RzcobsError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
