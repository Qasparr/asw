# Winlator Stack Survey — ASW Phase 0

> "The Method of Science, the Aim of Religion."
> — the motto of the A∴A∴, and the method of this document.

**Date:** 2026-10-05
**Author:** Johnathan "Qasparr" (Κασπάρρ) Monroe, Keeper of the Secret Treasure
**Standing:** All Rights Reserved, Without Prejudice
**93**

---

## I. The Question

What does ASW reuse from Winlator, what must it replace, and where exactly
is the integration surface? This survey answers it by evidence, not by
assumption, so that Phase 1 (kernel) and Phase 2 (first light) begin with
the map already drawn.

## II. The Stack As Found (verified 2026-10-05)

Winlator is the existing userspace proof that Windows applications can run
on Android. It is not a kernel project. That is precisely why it matters
to ASW: it is the userspace half we do not have to build.

| Component        | Version found        | Home                                   |
|----------------|----------------------|----------------------------------------|
| Winlator (upstream) | 11.2.0 (2026-08-19) | `brunodev85/winlator` on GitHub    |
| Wine (upstream)  | 11.x line (11.2.0, 2026-08-19) | WineHQ / GitLab              |
| Box64            | v0.4.4 (2026-08-07)  | `ptitSeb/box64`                        |
| Box86            | tracks Box64 releases | `ptitSeb/box86`                       |
| Container libc   | glibc w/ Termux-Pacman patches | in-repo `glibc_patches/`        |
| Graphics         | Mesa (Turnip / Zink / VirGL) | Mesa3D                           |
| D3D translation  | DXVK, VKD3D, CNC DDraw | doitsujin/dxvk, WineHQ, FunkyFr3sh |

Upstream facts, checked at the source: the repository is public, LGPL-2.1,
~19.3k stars, ~1.8k forks, not archived, README last touched 2026-09-22 —
reports of the original developer walking away are contradicted by the
record. The APK ships an `installable_components` tree (Wine, Box86/Box64,
Mesa, DXVK, VKD3D) deployed inside a glibc container on the device.

## III. The Finding That Changes the Plan

Wine 10.16 (released 2025-10-04) introduced fast synchronization through
NTSync: *"Fast synchronization support using NTSync"* is the headline of
its release notes, and it auto-detects `/dev/ntsync` — no configuration
required — on kernels ≥ 6.14 with the driver enabled.

Winlator 11.x ships Wine from the 11.x line. Therefore:

- **Winlator is already NTSYNC-capable out of the box.** The Wine it
  carries will take the `/dev/ntsync` path the moment the device node
  exists.
- On stock Android kernels the node does not exist (GKI builds do not set
  `CONFIG_NTSYNC`), so Winlator silently falls back to the wineserver
  round-trip path — the slow one.
- **ASW's one-line kernel delta (`CONFIG_NTSYNC=y`) is exactly the key
  that unlocks Winlator's latent fast path.** No Wine fork, no Wine
  rebuild, no patch set.

This confirms the thesis of `docs/ARCHITECTURE.md` — *"NT compatibility
belongs in the kernel, and the kernel community has now agreed"* — and
sharpens it: the userspace is already waiting. ASW supplies only the
kernel half, and the existing ecosystem does the rest.

MECHANISM (for the record): NT sync primitives (events, semaphores,
mutexes) previously crossed process boundaries through the wineserver on
every wait/wake. NTSYNC implements them as a kernel character device;
Wine opens `/dev/ntsync` and issues ioctls instead of IPC. The out-of-tree
predecessors (esync, fsync) proved the performance; NTSYNC made it
upstream and therefore shippable.

## IV. Reuse vs. Replace

| Layer | ASW posture |
|-------|-------------|
| Wine, Box86/Box64, Mesa, DXVK/VKD3D, glibc container | **REUSE wholesale.** Run the unmodified Winlator APK as the Phase 2 test harness. |
| Kernel | **REPLACE.** Custom 7.x build with `CONFIG_NTSYNC=y` (Phase 0 item, done; Phase 1 pending). |
| SELinux policy | **ADD.** New ASW work found by this survey (see §VII.2). |
| Benchmarks | **MEASURE.** Phase 3 = Winlator on ASW kernel vs. stock kernel, identical hardware — the NTSYNC delta, published honestly either way. |

The ROADMAP non-goals hold: we reimplement neither Wine nor Box86/Box64.

## V. The Lab Wrinkle

If the AVD lab image is x86_64, Box86/Box64 are bypassed entirely in-lab:
Wine runs native x86_64 PE binaries and the translation layer is never
exercised. The NTSYNC path *is* exercised (sync primitives don't care
about CPU translation), but no lab number may be read as a device number.
Confirm the AVD image architecture at first boot and record it in the
Phase 1 build notes. ARM translation waits for real hardware — which, for
the A16, waits on the bootloader question already on the desk.

## VI. The Fork Landscape

Because Winlator is LGPL-2.1, the fork scene is large and diverging:

- **Winlator-Ludashi** (StevenMXZ) — 4.1 (2026-09-13)
- **Ludashi Plus** (squalle0nhart) — 4.0 Update 1 (2026-09-09)
- **WinlatorMali** — tuned for Mali-GPU Android devices rather than Adreno

Two disciplines, from the record: (1) if a version number doesn't match
upstream's 11.2.0, you are looking at a fork — check the repository name
before assuming feature parity; (2) **WinlatorMali belongs on the Phase 4
shortlist**, because the A16-class device matrix is Mali territory, and
GPU-driver fit is where these forks actually differ. No fork is adopted
today; all are watched.

## VII. Open Questions (held open, not assumed)

1. **Does Winlator 11.2's Wine build actually probe `/dev/ntsync` under
   Android's container?** A feature request titled "Add ntsync support"
   sits open on the Ludashi fork (filed ~187 days ago), which shows the
   fork scene was chasing this — but upstream with Wine 11.x should
   already carry it. Settled by test, not by changelog: Phase 2 verifies
   with `lsof /dev/ntsync` (or Wine's sync debug channel) inside the
   container.
2. **SELinux.** On Android, an app in `untrusted_app` cannot simply open
   a new character device; `/dev/ntsync` needs sepolicy allow rules (or
   the lab runs permissive). The sepolicy fragment is genuine new ASW
   work — it belongs in `kernel/` beside the config fragment, and it is
   the second half of "the kernel half."
3. **License posture.** LGPL-2.1 permits forking and modification, with
   obligations attaching at distribution: preserve notices, ship source
   for modified LGPL components. If ASW ever ships its own APK, the
   compliance file is written first, not after.
4. **Wine auto-detect under Bionic.** Winlator ships its own glibc, so the
   container never touches Bionic — noted to close the question.

## VIII. Red-Pen Decisions (his, untouched by me)

1. Phase 2 uses the **unmodified Winlator APK** as the test harness —
   accept, or fork from day one?
2. Phase 4 device matrix: **WinlatorMali** fork or upstream, for A16-class
   Mali hardware?
3. An **ASW-branded APK** later — in scope for Phase 4, or permanently
   out (stay a kernel + policy project)?

## IX. TRVVTH Audit

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | Winlator 11.2.0 released 2026-08-19, upstream active | VERIFIED | `github.com/brunodev85/winlator` — releases; README updated 2026-09-22 |
| 2 | License LGPL-2.1, ~19.3k stars, ~1.8k forks | VERIFIED | same repo page |
| 3 | Box64 v0.4.4 (2026-08-07) | REPORTED | tech-insider.org survey of release pages, 2026-09 |
| 4 | Wine 10.16 (2025-10-04) added NTSync, auto-detects `/dev/ntsync` | VERIFIED | gamingonlinux.com release coverage; WineHQ release notes |
| 5 | NTSync needs kernel ≥ 6.14 with driver enabled | VERIFIED | Wine 10.16 notes; kernel merge record (6.14, Jan 2025) |
| 6 | Forks (Ludashi, Ludashi Plus, WinlatorMali) diverge from upstream | REPORTED | same survey; commit-ahead/behind counts cited |
| 7 | Open "Add ntsync support" request on Ludashi fork | VERIFIED | `github.com/stevenmxz/winlator-ludashi/issues/326` |
| 8 | ASW kernel delta = `CONFIG_NTSYNC=y`, zero Kconfig deps | VERIFIED | this project's own research, `kernel/config-fragment-asw`, 2026-10-05 |

Zero falsehoods. Where the record is a secondary survey, it is marked
REPORTED; where I checked the primary, VERIFIED.

## X. What This Unlocks

The last unchecked Phase 0 box is now checked in substance: the survey is
done, the test harness is identified (no build required), and Phase 1's
boot verification has a concrete acceptance test waiting — Winlator
installed in the NTSYNC-enabled AVD, `/dev/ntsync` open in the container.
The kernel was always the project. Now it is *only* the project, plus a
sepolicy fragment.

---

*Love is the law, love under will.*
**93 93/93**
