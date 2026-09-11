# P1 — PCIe DMA/Compute Accelerator Driver: Milestone Writeup (M0–M11)

This documents the full build-out of P1, from first PCI enumeration to a device that executes
real compute (not just memory copies) driven by a C++ userspace runtime. Every milestone below
was verified against a real (QEMU-emulated) device — not just written and assumed correct. Where
a milestone exposed a real bug, that's documented too: how it was found, what it turned out to
be, and how it was fixed, not just "it works now."

Companion documents: `README.md` (build instructions, repo layout) and
`dma-accel-v0-register-spec.md` (the authoritative register/ABI reference — every design
decision's full rationale lives there; this writeup is the narrative version).

## Overview

The accelerator is a custom PCIe device (`hw/misc/dma_accel.c` in a QEMU device model) that
started as a pure DMA copy engine and grew, milestone by milestone, into something that executes
real compute — a vector scale-add and a tile-level matrix multiply-accumulate. The stack has
three layers, each built and verified before the next depended on it:

```
QEMU device model  →  Linux kernel driver  →  C++ userspace runtime  →  GEMM workloads
  (dma_accel.c)        (dma_accel_drv.c)       (Stream/Buffer/Fence)
```

## M0–M2: PCIe enumeration, BAR0/MMIO, device identity

The driver's `probe()` maps BAR0, reads a `DEVICE_ID` register, and confirms it matches the
expected value before doing anything else. Trivial-sounding, but it's the first real proof the
guest kernel can even see the device correctly — everything downstream depends on this working.
**Verified**: `dmesg` shows `BAR0 mapped at <addr>, DEVICE_ID=0xd3a00001` and `DEVICE_ID verified
OK (M2 milestone passed)` on every successful boot.

## M3–M5: DMA transfer, IRQ delivery, error handling

A single-command DMA path: write source/dest addresses and length to registers, trigger the
transfer, wait for an MSI interrupt, read the result back. M4/M5 added the error paths this
needs to be trustworthy rather than just a happy-path demo:

- **Busy rejection**: submitting a second command while one is in flight is rejected
  synchronously (`ERR_BUSY`), and the original transfer still completes correctly — proving the
  rejection doesn't corrupt in-flight state.
- **Invalid length / invalid opcode**: rejected synchronously with the right error code, with no
  spurious `BUSY` state transition in between.

**Verified**: `dmesg` — `M3 self-test PASSED: 256 bytes copied correctly via DMA (IRQ path)`,
plus both M4 rejection cases confirmed correct on every boot.

## M6–M8: Command queue, concurrency, reset/recovery

M3–M5's one-command-at-a-time model doesn't scale. M6 replaced it with a proper submission/
completion ring-buffer pair (SQ/CQ, depth 16) that the device drains asynchronously — the driver
enqueues descriptors and gets notified via completions, instead of blocking on each one.

- **M7 (concurrency)**: 6 commands submitted back-to-back against a device that only executes 4
  concurrently (`MAX_INFLIGHT=4`). Verified genuine overlap, not serialization — completions
  arrive in two ~50ms-apart waves matching 4-then-2 execution, not six 50ms-apart singles.
- **M8 (reset/recovery)**: commands in flight or still queued when `RESET` hits get a distinct
  `ABORTED` completion status (not silently dropped, not conflated with a real error), and the
  device is fully usable again immediately after — verified by submitting a fresh, real transfer
  right after reset and confirming it completes correctly.

**Verified**: `dmesg` — `M6 self-test PASSED: 256 bytes copied correctly via SQ/CQ`, M7's two-wave
timing pattern, M8's 6/6 correctly-ABORTED commands plus a clean post-reset recovery check, on
every boot.

## M9: Userspace ABI

Opened the SQ/CQ mechanism to userspace: a char device (`/dev/dma_accel0`) with
`ioctl(BUFFER_ALLOC)` + `mmap()` for DMA-coherent buffers the CPU can read/write directly, and
`ioctl(SUBMIT)` + `poll()`/`read()` for command submission and completion. Two design decisions
worth calling out (both because they were deliberate, and because getting them wrong is a classic
category of bug):

- **`cmd_id` is kernel-allocated, not caller-supplied** (`atomic64_t` counter, one per device) —
  specifically to prevent two concurrent submitters from colliding on the same tag, which would
  silently misroute a completion to the wrong caller.
- **`read()` is always non-blocking and can return multiple completions per call** (not "one
  completion per `read()`") — the driver drains as many as fit in the caller's buffer. Getting
  this wrong (assuming one-at-a-time) would silently lose completions under load.

**Verified**: `dma_accel_test.cpp` — full `open`/`ioctl(BUFFER_ALLOC)`/`mmap`/`ioctl(SUBMIT)`/
`poll`/`read` chain, real data copied correctly, on real hardware.

## M10: C++ runtime (`Stream` / `Buffer` / `Fence`)

Lifted the raw ioctl ABI into a small, ergonomic C++ API: `Stream` owns the device fd and
completion processing, `Buffer` is an RAII-managed DMA-coherent allocation, `Fence` identifies
one command's completion state (a plain state object — no mutex, no thread; all processing
happens in `Stream::pump()`, called explicitly). All three types are move-only.

**Bug found and fixed here**: the driver's SQ ring has no full-queue detection —
`dma_accel_queue_submit()` unconditionally writes to `sq_tail % QUEUE_DEPTH` with no bounds
check, so submitting more than `QUEUE_DEPTH` (16) commands without any completing silently wraps
the ring and overwrites an unconsumed descriptor. M6–M9 never exceeded a handful of outstanding
commands, so this never surfaced. Rather than change the already-verified kernel queue path,
`Stream::submit()` now enforces the bound in userspace: once 16 are outstanding, the next
`submit()` blocks until the FIFO-oldest one completes. **Verified** with a dedicated stress test:
24 back-to-back submits (past the depth-16 boundary), all 24 completed with correct data, no
corruption.

## M11: Tiled GEMM correctness (three independently verified stages)

Before trying to make the device compute anything, proved the *tiling logic itself* is correct
— using the device purely as RAII-managed memory, no device-side compute yet:

1. **Stage 1 (indexing)**: A(256×128) × B(128×256) = C(256×256), K exactly one tile (no
   accumulation needed yet), all dimensions divide evenly (no boundary handling needed yet) —
   isolates whether the tile index arithmetic itself is right.
2. **Stage 2 (K-accumulation)**: same shape but K=256 (2 K-tiles), so each C tile now sums two
   (A-tile, B-tile) contributions instead of computing exactly one.
3. **Stage 3 (non-divisible dimensions)**: M=300, K=200, N=260 against tile=128 — none of the
   three dimensions divide evenly. Solved via zero-padding boundary tiles to full size rather
   than branching the compute loop, so the multiply-accumulate arithmetic itself never changes
   across all three stages.

Each stage deliberately added exactly one new variable over the last — never testing indexing,
accumulation, and boundaries all at once. **Verified**: all three stages, on real hardware,
match an independent untiled reference GEMM exactly, element-for-element.

## First real device compute: `OPCODE_SCALE_ADD`

Every milestone through M11 stage 3 used the accelerator purely as fast, shared memory — the CPU
did all the arithmetic. `SCALE_ADD` (`out[i] = a[i]*scalar + b[i]`) is the first opcode where the
*device* computes something. Chosen as the simplest possible "real compute" step deliberately —
no reduction, no control flow, so a bug can't hide behind arithmetic complexity.

Required growing the command descriptor from 40 to 48 bytes (`src2_addr` + `scalar` added) — the
first ABI-breaking change since M9. Verified in stages: local unit test against the device
model's execution function directly (via a fake guest-memory harness, since no real QEMU build
was available in that testing context), then a full regression of every prior test program
against the new 48-byte layout (confirming no COPY-path regression), then the real device.

**Follow-on proof**: chained 8 `SCALE_ADD` calls, each depending on the previous one's
completion, to compute one real row of `C = A @ B` (`acc[k] = B[k][:]*A_row[k] + acc[k-1]`).
This deliberately doesn't scale to a full GEMM (the math: a full M×K-reduction GEMM via this
row-chain approach costs roughly M×K serial submissions — impractical against a fixed ~50ms
per-command simulated latency) — its purpose was narrowly to prove the *chaining* is correct,
not that this is how a full GEMM should be computed on this device.

## Second real compute opcode: `OPCODE_TILE_MATMUL`

The row-chain math above made the real bottleneck explicit: per-element or per-row compute ops
don't scale, because submission count stays proportional to total FLOPs. `TILE_MATMUL` computes
a full 32×32 tile matmul-accumulate (`C_tile += A_tile @ B_tile`) in a *single* command — the
piece that actually makes device compute viable at real GEMM scale (one command replaces 32
chained `SCALE_ADD` calls' worth of accumulation). Tile size (32×32, not 128×128) was chosen
specifically to fit within the existing per-command byte cap without growing that device-wide
constant. Required no further descriptor growth — reuses `src_addr`/`src2_addr`/`dst_addr` as
A/B/C, with `dst_addr` read (the existing accumulator) *and* written (the new value), which is
what makes accumulation across a K dimension possible.

### The bug this one exposed

First real-hardware run produced results off by roughly 30 orders of magnitude — not a plausible
wrong answer, a sign of reading garbage memory. Diagnosed with temporary instrumentation printing
the state at each stage of the device model's execution function, which pointed straight at
`src2=0x0`: the device was reading the second input tile from guest physical address zero
instead of the real buffer.

Root cause was in the kernel driver, not the device model: resolving `src2_buffer_id` into a
real DMA address was gated on `opcode == OPCODE_SCALE_ADD` specifically — a leftover from when
`SCALE_ADD` was the only opcode needing a second input buffer. `TILE_MATMUL` matched neither that
check nor `COPY`, so the address silently stayed at its zero-initialized default. This is a
textbook example of why an *explicit, positive* enumeration of which opcodes need a resource beats
an implicit "not X" check — the fix replaced the opcode-specific gate with `dma_accel_opcode_needs_src2()`,
a small switch that must be updated by name for any future opcode needing `src2`, rather than
silently defaulting one way or the other. **Verified** post-fix: correct results, including a
specific check that the tile *accumulates* onto a nonzero starting value rather than overwriting
it (a starting accumulator with a nonzero, non-symmetric pattern — a bug that only overwrites
would still pass a zero-initialized test).

## Engineering practices worth calling out

- **Every milestone verified against real (emulated) hardware, not just compiled.** Several bugs
  in this project (the SQ backpressure gap, the `src2` resolution bug) only existed as a gap
  between "looks correct" and "is correct" — code review alone would not have caught either.
- **Regression discipline**: any change to the shared descriptor format or driver ABI triggers a
  full re-run of every existing test program, not just the new one — this is how the SQ
  backpressure fix and the 48-byte descriptor growth were both confirmed not to have broken
  anything already working.
- **One new variable at a time**: M11's three GEMM stages, and the SCALE_ADD → TILE_MATMUL
  progression, were each scoped to isolate exactly one untested thing rather than combining
  several unverified changes and hoping a single end-to-end pass covers all of them.
- **Local pre-verification before round-tripping to the VM**: several device-model changes were
  unit-tested against a minimal stub/fake-memory harness first, specifically to catch bugs before
  the slower cycle of rebuilding QEMU, restarting the VM, and re-testing on real hardware.
- **Tooling correctness matters too**: an incremental-build gotcha (`ninja` silently skipping a
  recompile when a copied-in file's mtime wasn't newer than the last build) once caused a test
  run against stale device-model code with no error or warning — worth remembering as a category
  of failure distinct from an actual logic bug.

## Deliberately not done (and why)

- **`BUFFER_FREE` / `GET_VERSION` ioctls**: no workload has needed mid-session buffer release or
  version negotiation yet — added only if one does.
- **A device-side `COPY` access pattern for GEMM tiling**: `Buffer` memory is already
  CPU/device-shared, so there's never been a real reason to stage data through an extra copy.
- **Scaling `SCALE_ADD`/`TILE_MATMUL` to a full production-size GEMM**: both compute opcodes have
  been proven correct at small, deliberately-scoped sizes (a proof of mechanism), not driven all
  the way through a full multi-tile GEMM workload — that's a distinct, larger effort from
  "does this opcode work at all."
