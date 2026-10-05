# ASW kernel build notes

## Base tree

`~/workspace/avd/kernel/linux-7.3-rc5/` — Linux 7.3-rc5 (VERSION=7,
PATCHLEVEL=3), the project's custom kernel target. NTSYNC merged upstream
in 6.14, so 7.3 carries it in-tree at `drivers/misc/ntsync.c`.

## Config research (done 2026-10-05)

`config NTSYNC` is a bare `tristate` with **no `depends on`** — the entire
ASW kernel delta is one line:

```
CONFIG_NTSYNC=y
```

`=y` (built-in) rather than `=m`: `/dev/ntsync` exists at boot, no module
loading needed in the AVD or on device. Fragment:
`kernel/config-fragment-asw` in this repo.

## Apply procedure (AVD target)

The 7.3-rc5 AVD build (`ARCH=arm64`, `aarch64-linux-gnu-gcc`) was already
mid-compile when ASW started — **do not interrupt it**. Apply after it
finishes; kbuild then only compiles `ntsync.c` and relinks (minutes):

```sh
cd ~/workspace/avd/kernel/linux-7.3-rc5
ARCH=arm64 scripts/kconfig/merge_config.sh \
    .config ~/workspace/asw/kernel/config-fragment-asw
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
```

## Verification

Boot the AVD image (see `~/workspace/avd/launch-qemu-aarch64.sh`,
KVM-capable host required) and check:

```sh
ls -l /dev/ntsync
```

Present = ASW kernel layer live. Absent = config didn't take; re-check
`.config` for `CONFIG_NTSYNC=y`.

## What this does not yet cover

- Wine + Box86/Box64 userspace stack (Phase 2).
- GKI/Android-specific kernel requirements beyond NTSYNC — to be
  researched when the device (not AVD) target firms up.
- Further NT-primitive extensions: only on measured need (see
  `docs/ARCHITECTURE.md`).
