# Running the QEMU boot test on Windows

**Yes — this runs natively on Windows.** The boot is pure software emulation:
`qemu-system-aarch64` (TCG) emulates an ARM64 `virt` board, and the kernel +
payload are just bytes loaded into RAM before the CPU starts. Neither XNU nor
the payload knows or cares what the host operating system is — Windows, Linux
and macOS all run the identical emulation with identical results.

The one thing that had to change for Windows: QMP (the monitor socket used to
stop the VM and snapshot the framebuffer) used a POSIX unix socket, which does
not exist on Windows. `tools/qemu-sweet.py` now picks a TCP loopback port
automatically on Windows, so no flags are needed. On POSIX everything behaves
exactly as before.

---

## 1. Install (one time)

| Tool | Install | Notes |
| --- | --- | --- |
| Python 3.8+ | `winget install Python.Python.3.12` or python.org | only stdlib is used |
| QEMU | `winget install SoftwareFreedomConservancy.QEMU` or <https://www.qemu.org/download/#windows> | add the install dir (e.g. `C:\Program Files\qemu`) to `PATH` |
| LLVM (optional) | `winget install LLVM.LLVM` | needed only for `--kernel` framebuffer text decoding and for building the userspace init |

Verify in PowerShell (reopen the terminal after installing so `PATH` refreshes):

```powershell
python --version
qemu-system-aarch64 --version
python tools\qemu-sweet.py --check-env
```

## 2. Build the test payload

Same commands as Linux (PowerShell syntax):

```powershell
# QEMU-compatible Apple Device Tree
python tools\mkadt.py payload.adt --qemu

# Pack kernel + ADT (+ optional userspace init ramdisk) into a SWEETXNU payload
python tools\mkpayload.py kernel.development.sweet payload.adt payload.bin `
    --cmdline "debug=0xc serial=3 cpus=1 keepsyms=1 rd=md0"
```

The kernel comes from the `xnu.yml` GitHub Actions build (see README
"Build the kernel") — download the `xnu-kernel` artifact and use
`kernel.development.sweet` from it.

To also boot the tiny userspace init (the `rd=md0` PID 1 demo), build it once
with the LLVM tools:

```powershell
cpp -x assembler-with-cpp -P tools\sweet-init.S -o sweet-init.s
llvm-mc -arch=arm64 -filetype=obj -triple=arm64-apple-ios16.0.0 sweet-init.s -o sweet-init.o
python tools\mkinit.py sweet-init.o init
python tools\mkpayload.py kernel.development.sweet payload.adt payload.bin `
    --ramdisk init --cmdline "debug=0xc serial=3 cpus=1 keepsyms=1 rd=md0"
```

## 3. Boot it

```powershell
python tools\qemu-sweet.py payload.bin out --secs 60 --kernel kernel.development.sweet --live-serial
```

Outputs in `out\`: `fb.png` (the 1080x2400 panel framebuffer — the Apple logo /
console), `fb.txt` (decoded console text), `serial.txt`, `regs.txt`,
`qemu.log`, `result.json`.

**Only for portable/extracted QEMU installs** whose compiled-in data directory
does not resolve (error mentioning `efi-virtio.rom` or other ROM files), point
QEMU at its bundled ROM directory explicitly:

```powershell
python tools\qemu-sweet.py payload.bin out --qemu-datadir "C:\Program Files\qemu\share\qemu"
```

Normal QEMU Windows installers do not need this.

## 4. What needs Windows' Linux layer (WSL2)

`tools/mkiosroot.sh` — building the real iOS 16.7.16 root filesystem from the
Apple IPSW — requires Linux host tools (`apfs-fuse`, `mkfs.hfsplus`/hfsprogs,
`rsync`, sudo, loop mounts). On Windows run that step inside **WSL2 (Ubuntu)**:

```bash
sudo apt update && sudo apt install hfsprogs rsync
```

`apfs-fuse` is not in Ubuntu's archive; build it once:

```bash
sudo apt install git cmake libfuse3-dev libicu-dev bzip2
git clone https://github.com/sgan81/apfs-fuse && cd apfs-fuse
git submodule init && git submodule update
mkdir build && cd build && cmake .. && make -j && sudo make install
```

Then run `tools/mkiosroot.sh ...` as in the README, and either boot inside
WSL2 (its QEMU is fine) or copy the finished image to Windows and boot it with
the Windows QEMU — the payload only embeds the image bytes.

## 5. Summary of host-OS support

| Step | Windows native | WSL2 | Linux / macOS |
| --- | --- | --- | --- |
| `mkpayload.py`, `mkadt.py`, `mkinit.py` (payload build) | yes | yes | yes |
| `qemu-sweet.py` boot + Apple logo + framebuffer PNG | yes (TCP QMP auto) | yes | yes |
| `--kernel` framebuffer text decoding (`llvm-nm`) | yes (LLVM installed) | yes | yes |
| `mkiosroot.sh` real iOS rootfs from the IPSW | no — needs Linux | **yes** | yes |
| KVM/HVF acceleration | no (TCG everywhere) | no (nested) | yes (Linux KVM, macOS HVF) |
