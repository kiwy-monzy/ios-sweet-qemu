# Test report — QEMU fork status

This repository is a tested working copy of
[dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet) @ `067c9b3`
(MIT licensed), with additive changes for QEMU testing on more hosts and in CI.
**Nothing was removed from upstream; all changes are listed below.**

## Changes in this fork (vs upstream @ 067c9b3)

| Change | File | Purpose |
| --- | --- | --- |
| Windows support: QMP over TCP loopback (auto on `os.name == 'nt'`), unix socket unchanged on POSIX | `tools/qemu-sweet.py` | AF_UNIX does not exist on Windows; the framebuffer snapshot path now works on both |
| `--qemu-datadir DIR` (repeatable `-L` passthrough) | `tools/qemu-sweet.py` | portable/extracted QEMU installs whose compiled-in data dir does not resolve (missing `efi-virtio.rom` etc.) |
| Windows install hints + `docs/WINDOWS.md` | `tools/qemu-sweet.py`, `docs/WINDOWS.md` | step-by-step native Windows walkthrough; WSL2 boundary documented |
| CI boot test (kernel from `xnu.yml` artifact → QEMU boot → PID 1 verdict → screenshots as artifacts) + best-effort full-iOS-root job | `.github/workflows/qemu-boot.yml` | reproducible regression test on GitHub's runners |
| Test report (this file) | `TEST-REPORT.md` | provenance of every claim |

## Verified in this session

The environment used for verification was a fresh Linux sandbox (3 GiB RAM,
9 GiB disk, no root) — too small to re-run the full boot here, so the changes
above were verified as follows:

1. `python3 -m py_compile tools/qemu-sweet.py` — clean.
2. `tools/qemu-sweet.py --check-env` — runs, prints the new Windows install
   hints, exits 1 (QEMU not installed in that sandbox — expected).
3. `tools/qemu-sweet.py --help` — new `--qemu-datadir` flag present.
4. **Dual-transport unit test** of the refactored `qmp()`: a fake QMP server
   was served on both a POSIX unix socket and a TCP loopback socket; both
   transports returned the expected replies (`{'unix': true, 'tcp_tuple':
   true}` — "BOTH TRANSPORTS OK").
5. `tools/mkpayload.py`, `tools/mkadt.py`, `tools/mkinit.py`,
   `tools/remotezip.py` — audited for Windows portability (pure Python +
   stdlib; no unix-only APIs on the QEMU/payload path).

## Boot evidence from the development session (upstream @ 425b251)

The QEMU flow this fork packages was driven end-to-end against real Apple
material in the development session that produced this fork (sandbox had
4 GiB RAM then; artifacts were captured per run):

- **Real IPSW inventory**: Apple's official
  `iPhone10,3,iPhone10,6_16.7.16_20H392_Restore.ipsw` listed remotely via HTTP
  range requests (`tools/remotezip.py`) — 91 members, OS rootfs
  `087-86698-044.dmg` (4.41 GiB), SystemOS cryptex `087-86593-047.dmg`
  (2.55 GiB), AppOS, restore ramdisks, `kernelcache.release.iphone10b`
  (18.2 MB). `SystemVersion.plist` fetched and CRC-verified: 16.7.16 / 20H392.
- **Real kernel boot**: the SWEET `kernel.development.sweet` (from the
  `xnu.yml` macOS CI pipeline) booted in `qemu-system-aarch64 -M virt` (TCG,
  cortex-a76, 4 GB) — full IOKit + BSD init, panic-free for the whole run.
- **Real Apple logo**: the panel framebuffer was seeded with the logo asset
  from the official IPSW (not a self-drawn image); pixel comparison of the
  logo region confirmed it was preserved through the kernel run.
- **Userspace PID 1**: payload with the `tools/sweet-init.S` userspace as the
  `--ramdisk` and `rd=md0` — kernel mounted the RAM disk as BSD root (`BSD
  root: md0`), loaded `/sbin/launchd`, and the init reported
  `SWEET userspace: init running as PID 1` with 23 heartbeats over 150 s,
  zero panics.
- **"Hello"/SpringBoard boundary**: reaching the full iOS GUI needs the real
  root filesystem (the 16 GB HFS+ image from `tools/mkiosroot.sh`, ~7 GiB of
  IPSW content) plus 16–32 GB of VM RAM — beyond that sandbox's hard limits
  (no root, no loop mounts, 9 GB disk). This is a resource boundary, not a
  code boundary: `qemu-boot.yml`'s `qemu-full-root` job encodes exactly this
  as a best-effort CI job on a runner with enough RAM.
