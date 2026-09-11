# ai-accelerator-stack

A custom PCIe DMA/compute accelerator, built end-to-end: QEMU device model → Linux kernel driver
→ userspace ABI → C++ runtime → GEMM workloads. Everything below has been verified on real
(emulated) hardware, not just written — see [Status](#status) for exactly what's been run and
what hasn't.

`dma-accel-v0-register-spec.md` is the authoritative register/ABI contract this whole stack is
built against — driver, device model, and runtime must all match it byte-for-byte. This README
is the entry point and build guide; the spec doc is where the actual design decisions, their
rationale, and the full verification history live.

## Scope of this README

This covers everything from "you already have a QEMU source checkout and a bootable guest VM
image" onward — cloning/configuring QEMU itself, registering a new PCI device in QEMU's build
system (`hw/misc/meson.build` / `Kconfig`), and building the guest VM image (this project uses
an Ubuntu 24.04 cloud image + a `cloud-init` seed ISO) are **not** documented here, because
they're one-time environment setup this repo's own history doesn't include step-by-step. If
you're reproducing this from zero, budget time for that setup separately — the instructions
below assume it's already done.

## `dma_accel.c` lives in two places — know which one is the source of truth

This repo's `dma_accel.c` is the QEMU device model source and the one you edit and commit. It is
**not** where QEMU actually reads it from when building — QEMU compiles whatever's sitting at
`hw/misc/dma_accel.c` inside your separate QEMU source checkout. The relationship:

```
this repo's dma_accel.c          ←  edit here, commit here, source of truth
        │
        │  cp (+ touch — see the ninja gotcha below)
        ▼
qemu-src/hw/misc/dma_accel.c     ←  pure build artifact location, never edited directly,
                                     never committed (it's inside QEMU's own git history,
                                     which is a separate, unrelated repo you didn't create)
```

Don't edit the copy inside `qemu-src/` directly — changes made there will get overwritten the
next time you `cp` this repo's version in, and won't be captured in this repo's git history
either way. Don't add `qemu-src/` itself to this repo (it's a full QEMU checkout, hundreds of
thousands of files, with its own unrelated upstream git history) — the `.gitignore` here
guards against that if `qemu-src` ever ends up nested inside this repo by accident, but keeping
it as a sibling directory (`~/projects/qemu-src`, not `~/projects/ai-accelerator-stack/qemu-src`)
avoids the question entirely.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Guest VM (Ubuntu 24.04, QEMU)                                │
│                                                                │
│  Userspace                                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Test programs / GEMM workloads                        │   │
│  │        │                                               │   │
│  │        ▼                                               │   │
│  │ dma_accel_runtime.hpp/cpp  (Stream / Buffer / Fence)   │   │
│  │        │  open() / ioctl() / mmap() / poll() / read()  │   │
│  └────────┼──────────────────────────────────────────────┘   │
│  Kernel   ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ dma_accel_drv.c  — char device, SQ/CQ management,     │   │
│  │                     IRQ handling, buffer lifecycle     │   │
│  └────────┼──────────────────────────────────────────────┘   │
│  PCI      ▼ MMIO / DMA                                        │
└───────────┼─────────────────────────────────────────────────┘
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Host: QEMU process                                            │
│  hw/misc/dma_accel.c  — device model: registers, SQ/CQ        │
│  execution engine, MSI, opcode dispatch (COPY / SCALE_ADD /   │
│  TILE_MATMUL)                                                 │
└─────────────────────────────────────────────────────────────┘
```

`dma_accel_regs.h` is shared verbatim by the driver and the device model (and, for the userspace
ABI portion, by the runtime too) — it's the single place register offsets, opcode values, and
descriptor layouts are defined, precisely so the three implementations can't silently drift
apart.

## Repository layout

| File | Layer | Purpose |
|---|---|---|
| `dma_accel.c` | QEMU device model | Lives at `hw/misc/dma_accel.c` in a QEMU source tree |
| `dma_accel_drv.c` | Kernel driver | PCI driver + char device (`/dev/dma_accel0`) |
| `dma_accel_regs.h` | Shared header | Register offsets, opcodes, descriptor structs — included by both the driver and the device model |
| `dma_accel_runtime.hpp` / `.cpp` | C++ runtime | `Stream` / `Buffer` / `Fence` — the userspace API workloads actually call |
| `dma-accel-v0-register-spec.md` | Docs | Authoritative spec — design rationale, full verification history, every bug found and how it was fixed |
| `start-vm-dma-accel.sh` | Tooling | Launches the QEMU guest with the `dma-accel` device attached, a serial console log, and a QEMU monitor socket |
| `dma_accel_test.cpp` | Test | Raw uAPI smoke test (ioctl/mmap/poll/read directly, no runtime) |
| `dma_accel_runtime_smoke_test.cpp` | Test | Single COPY through `Stream`/`Buffer`/`Fence` |
| `dma_accel_runtime_stress_test.cpp` | Test | 24 back-to-back submits — exercises SQ-depth backpressure |
| `dma_accel_gemm_stage1/2/3.cpp` | Test | Tiled GEMM correctness: indexing → K-accumulation → non-divisible dimensions (all CPU-side math over device memory) |
| `dma_accel_scale_add_test.cpp` | Test | First real device-side compute (`out[i] = a[i]*scalar + b[i]`) |
| `dma_accel_gemm_row_via_scale_add.cpp` | Test | One GEMM row via 8 chained `SCALE_ADD` calls |
| `dma_accel_tile_matmul_test.cpp` | Test | Single-command 32×32 device-side tile matmul-accumulate |

## Prerequisites

Assumed to already exist before any of the steps below will work:

- A QEMU source checkout (this project has been built/tested against QEMU 8.2.2) with the
  `dma-accel` device already registered in the build system (`hw/misc/meson.build`/`Kconfig`) —
  see [Scope of this README](#scope-of-this-readme) above for why that registration step isn't
  documented here.
- A bootable Ubuntu 24.04 ("Noble") guest VM image, reachable via `ssh -p 2222 ubuntu@localhost`
  once `start-vm-dma-accel.sh` is running (this project's guest was set up via a cloud image +
  `cloud-init` seed ISO — the exact image/seed files aren't part of this repo).
- Kernel headers on the guest matching its running kernel (`uname -r`), needed to build the
  out-of-tree `dma_accel_drv.c` module — typically `sudo apt install linux-headers-$(uname -r)`
  on a stock Ubuntu cloud image, but verify against whatever's actually installed.
- A C++17 compiler (`g++`) on the guest for the runtime/test programs.

## Building

### 0. Get this repo

```bash
git clone git@github.com:Lengfeng88/ai-accelerator-stack.git
cd ai-accelerator-stack
```

Everything below assumes you're running from inside this directory, and that `qemu-src` (see
[above](#dma_accelc-lives-in-two-places--know-which-one-is-the-source-of-truth)) is a sibling
directory, not nested inside this one.

### 1. QEMU device model

```bash
cp dma_accel.c ~/projects/qemu-src/hw/misc/dma_accel.c
touch ~/projects/qemu-src/hw/misc/dma_accel.c   # see gotcha below
cd ~/projects/qemu-src/build
ninja
```

> **Gotcha:** `ninja` does incremental builds based on file mtime. If you `cp` or `scp` a file in
> and its mtime ends up not newer than the last build, `ninja` silently skips recompiling it —
> you'll be running stale device-model code with no error or warning. Always `touch` the file
> before `ninja` after copying it in from elsewhere. Confirm the build actually did something by
> checking the ninja output includes `Compiling C object ... hw_misc_dma_accel.c.o` and `Linking
> target qemu-system-x86_64` — if it's a single step (just `Generating qemu-version.h`), nothing
> was recompiled.

### 2. Start the VM

```bash
chmod +x start-vm-dma-accel.sh
nohup ./start-vm-dma-accel.sh </dev/null >/tmp/qemu-stdout.log 2>&1 &
disown
```

Guest console log: `/tmp/ai-accel-console.log`. QEMU monitor: `socat - unix:/tmp/ai-accel-monitor.sock`.
QEMU's own stdout/stderr (device-model `fprintf` debug output, if any is temporarily added):
`/tmp/qemu-stdout.log` — note this is a **host-side** file, not visible from inside the guest.

### 3. Kernel driver (inside the guest)

```bash
scp -P 2222 dma_accel_drv.c dma_accel_regs.h ubuntu@localhost:~/dma-accel-driver/
ssh -p 2222 ubuntu@localhost
cd ~/dma-accel-driver
make clean && make
sudo dmesg -C
sudo insmod dma_accel.ko
sudo dmesg -w
```

Watch for `dma-accel: /dev/dma_accel0 ready` after the M3–M9 in-kernel self-tests pass. The
module isn't retained across VM reboots — every fresh boot needs `insmod` again.

### 4. Runtime + test programs (inside the guest)

```bash
scp -P 2222 dma_accel_runtime.hpp dma_accel_runtime.cpp dma_accel_*test*.cpp ubuntu@localhost:~/dma-accel-driver/
g++ -std=c++17 -O2 -Wall -Wextra -I. -o dma_accel_tile_matmul_test dma_accel_tile_matmul_test.cpp dma_accel_runtime.cpp
sudo ./dma_accel_tile_matmul_test
```

Same pattern for every other `dma_accel_*test*.cpp` — each is a standalone translation unit
that links against `dma_accel_runtime.cpp` (except `dma_accel_test.cpp`, which talks to the raw
uAPI directly and has no runtime dependency).

**After any driver or device-model change, re-run the full test suite in order, not just the new
test** — several past bugs (an SQ-ring backpressure gap, a `src2` address-resolution bug) were
only caught because earlier, already-passing tests were re-run and either still passed
(confirming no regression) or newly failed (catching a regression immediately instead of much
later):

```bash
sudo ./dma_accel_test
sudo ./dma_accel_runtime_smoke_test
sudo ./dma_accel_runtime_stress_test
sudo ./dma_accel_gemm_stage1
sudo ./dma_accel_gemm_stage2
sudo ./dma_accel_gemm_stage3
sudo ./dma_accel_scale_add_test
sudo ./dma_accel_gemm_row_via_scale_add
sudo ./dma_accel_tile_matmul_test
```

## Opcodes

| Opcode | What it does | Single command does |
|---|---|---|
| `OPCODE_COPY` | `dst = src` | Byte copy, up to 4096 bytes |
| `OPCODE_SCALE_ADD` | `out[i] = a[i]*scalar + b[i]` | One float32 vector op, up to 1024 elements |
| `OPCODE_TILE_MATMUL` | `C_tile += A_tile @ B_tile` | One 32×32 matmul-accumulate |

Full descriptor layouts, register map, and design rationale for each: see `dma-accel-v0-register-spec.md` §8.1 and §11.3.

## Status

Everything below is implemented **and** verified against real (emulated) hardware — dmesg
output and/or passing test binaries exist for each, not just code that looks right:

- **M0–M2**: PCIe enumeration, BAR0/MMIO, device ID verification
- **M3–M5**: DMA transfer via IRQ, busy-rejection, invalid-length/opcode error paths
- **M6–M8**: SQ/CQ command queue, 4-way concurrent execution, reset/recovery
- **M9**: Userspace ABI (`ioctl`/`mmap`/`poll`/`read`) — raw, no runtime
- **M10**: C++ runtime (`Stream`/`Buffer`/`Fence`) — plus a discovered-and-fixed SQ-depth
  backpressure gap (the driver has no full-queue detection; the runtime enforces the bound)
- **M11**: Tiled GEMM correctness (indexing, K-accumulation, non-divisible dimensions) — CPU-side
  math over device-allocated memory, not device compute
- **Compute opcodes**: `SCALE_ADD` (single op, then chained across 8 steps to compute a real GEMM
  row) and `TILE_MATMUL` (single-command 32×32 matmul-accumulate) — the first two opcodes that
  make the accelerator actually compute, not just move bytes. Uncovered a real driver bug
  (`src2` address resolution silently defaulting to zero for a new opcode) in the process — see
  the spec doc's §11.3 for the full story and the fix.

**Explicitly not done, by choice, not oversight** (see the spec doc for why each was deferred):

- `BUFFER_FREE` / `GET_VERSION` ioctls — no workload has needed mid-session buffer release yet
- A device-side `COPY` access pattern for GEMM tiling — buffers are already CPU/device-shared
  memory, so no forcing function has appeared for actually moving data through a COPY
- Scaling `TILE_MATMUL` or `SCALE_ADD` to a full-size GEMM — chaining `SCALE_ADD` row-by-row was
  proven correct at small scale (K=8) specifically to establish the technique works, not as a
  practical way to run a real GEMM (the per-command latency makes that math genuinely
  impractical at real matrix sizes; `TILE_MATMUL` is the piece that actually makes device compute
  viable at scale, and hasn't yet been driven all the way through a full multi-tile GEMM)
