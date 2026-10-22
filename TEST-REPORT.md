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

## Verified on a native Windows host (2026-10-22)

The same boot was re-run on a real Windows 11 machine (i7-7700, 31.9 GiB RAM,
QEMU 11.0.93 for Windows, LLVM 20.1.7, Python 3.13.15) — no WSL, no Linux:

```
clang -x assembler-with-cpp -E -P tools\sweet-init.S -o sweet-init.s
clang -target arm64-apple-ios16.0.0 -c sweet-init.s -o sweet-init.o
python tools\mkinit.py sweet-init.o init
python tools\mkadt.py payload.adt --qemu
python tools\mkpayload.py kernel.development.sweet payload.adt payload.bin \
    --ramdisk init --cmdline "debug=0xc serial=3 cpus=1 keepsyms=1 rd=md0"
python tools\qemu-sweet.py payload.bin out --secs 150 --kernel kernel.development.sweet --live-serial
```

Result: the payload is byte-identical in layout to the CI build
(`virt_base 0xfffffe0006000000`, `entry 0xfffffe000748b488`,
`ramdisk 0xb24c8000`, `size 16621936`), and the boot reached
`BSD root: md0` → `load_init_program: attempting to load /sbin/launchd` →
`*** SWEET userspace: init running as PID 1 ***` with 23 heartbeats over
150 s and no panic. `out/fb.png` holds the framebuffer screenshot of that
run (the SWEET console is drawn into the panel, so the decoded console
text is in `out/fb.txt`).

Note: `out/serial.txt` stays **empty** in this configuration — the SWEET
kernel prints its console into the framebuffer, not the UART. Anything
that grades a boot must read `out/fb.txt` (see the CI fix below).

## CI status (2026-10-22) and the fix

Run `39776897329` (`qemu-boot`, job `qemu-userspace` = job id `119818017659`)
**booted correctly but was reported as failed**: the verdict step read
`out/serial.txt` (empty, see above) instead of the decoded framebuffer text,
so `bsd_root_md0`/`pid1` were `false` although `out/fb.txt` contained
`BSD root: md0` and `init running as PID 1` (17 heartbeats, no panic).

Job `qemu-full-root` (job id `119818017825`) failed even earlier: `apfs-fuse`
does not compile without `bzlib.h` (`libbz2-dev` was missing from the apt
list), and its informational verdict then died with
`FileNotFoundError: out/full-root-verdict.json` instead of writing the file.

Commit `6495b3a` fixes both: the verdict now reads `out/fb.txt` first and
falls back to `out/serial.txt` (matching a whitespace-flattened copy, because
the framebuffer wraps console lines), reports which source it used, always
writes its JSON, and the full-root job installs `libbz2-dev`, `zlib1g-dev`
and `pkg-config`. The new verdict was checked against the artifacts of run
`39776897329` and against the local Windows run: `pass: true` in both cases
(`bsd_root_md0=true`, `pid1=true`, `heartbeats=17/23`, `panic=false`).

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
