# ASW Architecture

## Why kernel-level

Wine translates Win32 API calls in userspace. For most calls that's fine.
Synchronization primitives — mutexes, semaphores, events — are the exception:
every wait/wake crossed process boundaries through the wineserver, and
multithreaded Windows applications choked on the round-trips. The old answers
were out-of-tree patches: esync, fsync. They worked, but they were hacks
carried outside the kernel.

NTSYNC (Linux 6.14, Jan 2025) ended that era: NT sync semantics implemented
as a proper kernel driver, upstream, no patches. Wine 11 uses it
automatically. The lesson ASW takes from this: **NT compatibility belongs in
the kernel**, and the kernel community has now agreed.

## WSL, inverted

| WSL (Microsoft)              | ASW (this project)                |
|------------------------------|-----------------------------------|
| Real Linux kernel on Windows | NT-compat kernel layer on Android |
| Lightweight Hyper-V VM       | Native execution, no VM           |
| Closed host, open guest      | Open host, closed guest API       |
| Goal: Linux apps on Windows  | Goal: Windows apps on Android     |

WSL2 ships a whole kernel because Windows couldn't host Linux otherwise.
ASW doesn't need a VM: Android *is* Linux. It needs the kernel to speak
enough NT that Wine stops translating and starts calling.

## The three layers in detail

### 1. Kernel layer (the ASW contribution)

- **NTSYNC** (`CONFIG_NTSYNC`): the merged driver. Baseline requirement.
- **Future work**: further NT-primitive surface as justified by measurement —
  each addition must prove it beats the userspace path, the way NTSYNC did.
  No speculative kernel code; the kernel stays lean.
- **Kernel version**: NTSYNC needs ≥ 6.14. Stock Android kernels (GKI) trail
  mainline, so ASW rides the project's custom kernel build (7.x target),
  where the version requirement is satisfied by construction.

### 2. CPU translation (Box86/Box64)

ARM phones can't execute x86 code. Box86 (32-bit) and Box64 (64-bit)
translate userspace x86→ARM64 dynamically. This layer is mature and
external — ASW integrates it, doesn't rewrite it.

### 3. API translation (Wine)

Wine maps the Win32 API surface onto POSIX. On an ASW kernel it takes the
fast path: sync objects go to `/dev/ntsync` instead of the wineserver.
Everything else in Wine is unchanged.

## What success looks like

A Windows PE binary launched on Android, with its synchronization running
through the kernel — measurably faster than the same binary under
Winlator's userspace-only stack on identical hardware. That measurement,
on a Galaxy A16-class device, is the project's first real milestone.
Anything before it is scaffolding.
