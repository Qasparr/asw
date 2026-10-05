# ASW Roadmap

## Phase 0 — Paper (current)

- [x] Vision and architecture written (`README.md`, `docs/ARCHITECTURE.md`)
- [x] Kernel config research: exact `CONFIG_` set for NTSYNC + deps on the
      target kernel tree — answer: one line, `CONFIG_NTSYNC=y`, zero
      dependencies (verified against 7.3-rc5 Kconfig 2026-10-05).
      Fragment: `kernel/config-fragment-asw`. Build docs: `docs/KERNEL.md`.
- [x] Survey Winlator's stack for integration points (what ASW reuses vs.
      what it replaces) — `docs/WINLATOR-SURVEY.md` (2026-10-05). Finding:
      Winlator 11.x ships Wine 11.x, which auto-detects `/dev/ntsync`;
      ASW reuses the whole userspace unmodified, replaces only the kernel,
      and adds a sepolicy fragment (new work item). Phase 0 complete.

## Phase 1 — Kernel

- [ ] Custom kernel (7.x target) built with `CONFIG_NTSYNC=y`
- [ ] Boot in the AVD lab; verify `/dev/ntsync` exists and answers
- [ ] Document the build reproducibly (config, toolchain, steps)

## Phase 2 — First light

- [ ] Box86/Box64 + Wine 11+ assembled on the ASW kernel
- [ ] First Windows PE "hello world" executed on Android
- [ ] Confirm Wine takes the NTSYNC path (not wineserver fallback)

## Phase 3 — Measurement

- [ ] Benchmark suite: multithreaded Windows workloads, ASW kernel vs.
      stock kernel + Winlator on identical hardware
- [ ] Publish numbers honestly — including the ones that don't favor us

## Phase 4 — Packaging

- [ ] Define what "installing ASW" means for a user (kernel + stack)
- [ ] Device support matrix, starting with Galaxy A16-class hardware

## Non-goals

- Reimplementing Wine or Box86/Box64.
- Shipping the NT kernel (closed-source; impossible).
- Supporting every Windows app on day one — correctness first, breadth later.
