# ASW — Android Subsystem for Windows

WSL in reverse. Windows Subsystem for Linux put a real Linux kernel inside
Windows. **ASW puts Windows application support into Android's kernel** — not
userspace API translation alone (that's Wine's job, and Wine already exists),
but kernel-level NT compatibility in the Android Linux kernel itself.

## The anchor

In January 2025, **NTSYNC** merged into mainline Linux (6.14): a kernel driver
(`drivers/misc/ntsync.c`, `/dev/ntsync`) implementing Windows NT
synchronization primitives — mutexes, semaphores, events — natively in the
kernel. Built by Valve's Elizabeth Figura for Wine/Proton. Wine 11 detects it
automatically and uses it when present; some workloads see triple-digit
percentage gains over the old wineserver path.

This is the precedent and the foundation: mainline Linux now speaks NT sync.
ASW carries that philosophy further on Android.

## Architecture

```
┌─────────────────────────────────────────────┐
│  Windows applications (x86/x86_64 PE)       │
├─────────────────────────────────────────────┤
│  Wine — Win32 API translation (userspace)   │
├─────────────────────────────────────────────┤
│  Box86 / Box64 — CPU translation (ARM host) │
├─────────────────────────────────────────────┤
│  ASW kernel layer                           │
│   · NTSYNC (/dev/ntsync)                    │
│   · future NT-primitive extensions          │
│   · Android kernel (GKI/custom, ≥ 6.14)     │
└─────────────────────────────────────────────┘
```

Three layers, three jobs:

1. **Kernel** — NT primitives where they belong: in the kernel. NTSYNC today;
   further NT syscall-surface work as the project matures. Requires kernel
   ≥ 6.14, which stock Android kernels largely don't ship yet — hence the
   custom kernel (see the 7.x kernel project).
2. **CPU translation** — Box86/Box64 turns x86/x86_64 code into ARM64 on the
   fly. No translation needed for the rare ARM64 Windows binary.
3. **API translation** — Wine maps Win32 to POSIX/Android. With the kernel
   layer beneath it, Wine stops emulating sync in userspace and talks to
   `/dev/ntsync` directly.

## What ASW is not

- Not a Windows kernel on Android. The NT kernel is closed-source and x86;
  that door is closed. ASW is NT *compatibility* in the Linux kernel, the
  same way WSL2 is a Linux kernel on Windows rather than Linux *emulation*.
- Not a Wine replacement. Wine is the userspace half; ASW is the kernel
  half. Winlator already proves the userspace stack works on Android —
  ASW gives it the ground it deserves.

## Status

**v0.1 — blueprint.** This repo currently holds the vision, the architecture,
and the kernel research. No kernel has been built yet; no device runs ASW.
Everything here is honest about that. See `docs/ARCHITECTURE.md` and
`docs/ROADMAP.md`.

## Roadmap

- [ ] Kernel config research: NTSYNC + dependencies on an Android GKI/custom
      kernel tree; document the exact `CONFIG_` set.
- [ ] Build the custom kernel (7.x target) with NTSYNC enabled.
- [ ] Boot it in the AVD lab; verify `/dev/ntsync` present.
- [ ] Wine + Box86/Box64 stack on top; first "hello world" PE on Android.
- [ ] On-device run (Galaxy A16 class hardware); measure vs. Winlator baseline.
- [ ] Packaging: what "installing ASW" looks like for a user.

## License

TBD by the author's red pen.
