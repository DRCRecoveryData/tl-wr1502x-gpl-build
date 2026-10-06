# GitHub-ready package

I'll give you four files: `README.md`, `build.sh`, `extract-rootfs.sh`, and `flash.sh`. Plus a `.gitignore` and `LICENSE` note.

The structure will be:

```
tl-wr1502x-firmware/
├── README.md
├── build.sh
├── extract-rootfs.sh
├── flash.sh
├── .gitignore
├── LICENSE
└── patches/
    └── (patched-tree.tar.bz2 goes here)
```

Create the directory and files:

```bash
mkdir -p ~/tl-wr1502x-firmware/patches
cd ~/tl-wr1502x-firmware

# Copy the patch tarball in
cp ~/tl-wr1502x-build/20261006/patched-tree.tar.bz2 patches/
```

## `README.md`

```bash
cat > README.md <<'ENDREADME'
# TL-WR1502X v1 — GPL Firmware Rebuild

[![Build](https://img.shields.io/badge/build-passing-brightgreen)]()
[![Platform](https://img.shields.io/badge/platform-MIPS%2024Kc-blue)]()
[![Kernel](https://img.shields.io/badge/kernel-4.4.176-orange)]()
[![License](https://img.shields.io/badge/license-GPL--2.0-lightgrey)]()

Reproducible build of TP-Link **TL-WR1502X v1** router firmware from the
official GPL source drop, ported to build on modern Linux (Ubuntu 24.04
+ GCC 15).

The SDK was written for CentOS 7 / Ubuntu 16.04 with GCC 4.9. This
repository patches every known incompatibility so it builds cleanly on
a 2026-era host.

---

## Hardware

| Component | Part |
|---|---|
| SoC | Realtek **RTL8197F VG** (MIPS 24Kc @ 400 MHz, dual-core) |
| Wi-Fi | Realtek **RTL8852CE** + RTL8832BR family, RFE54 front-end |
| Switch | Realtek **RTL8367R** |
| Flash | **80 MB SPI NAND** (`0x5000000`) |
| RAM | **128 MB** DDR2, auto-detected |
| Console | `ttyS0` @ **38400** 8N1, no flow control |

Bootloader and wireless calibration are pre-provisioned and not modified
by this build.

---

## Status

- ✅ Builds cleanly on Ubuntu 24.04 / GCC 15
- ✅ Produces flashable `fw.bin` (7.58 MB)
- ✅ All 100+ userspace packages compile
- ✅ Kernel + MIPS cross-compile
- ✅ Rootfs squashfs
- ⚠️ **Not tested on real hardware** (contributions welcome)

Reference build:
```
a22b083a7f2e4f4667f0b9b5167a614d  fw.bin
```

---

## Quick start

### Prerequisites

```bash
sudo apt update
sudo apt install -y \
  build-essential bison flex \
  libncurses5-dev libncursesw5-dev \
  zlib1g-dev libssl-dev bc rsync \
  texinfo git wget python2.7
```

You also need the MIPS cross-toolchain from the SDK tarball
(`rtl819x.tar.bz2`), which is **not included here** — download from
TP-Link's GPL portal.

### Layout

```
tl-wr1502x-firmware/
├── README.md
├── build.sh
├── extract-rootfs.sh
├── flash.sh
├── patches/
│   └── patched-tree.tar.bz2
└── (expected outside: GPL_TL-WR1502Xv1/rtl8197/ from the SDK tarball)
```

### Build

```bash
# 1. Extract the original GPL SDK somewhere
tar -xjf rtl819x.tar.bz2 -C ~/sdk/

# 2. Apply patches
./build.sh --apply-patches ~/sdk/GPL_TL-WR1502Xv1

# 3. Build (30–90 minutes)
./build.sh --sdk ~/sdk/GPL_TL-WR1502Xv1

# 4. Result
ls ~/sdk/GPL_TL-WR1502Xv1/rtl8197/target/image/fw.bin
```

Or run in one shot:

```bash
./build.sh --sdk ~/sdk/GPL_TL-WR1502Xv1 --apply-patches-from patches/patched-tree.tar.bz2
```

### Inspect the rootfs

```bash
./extract-rootfs.sh ~/sdk/GPL_TL-WR1502Xv1/rtl8197/target/image/root.bin /tmp/router
```

Then read:
```
/tmp/router/etc/init.d/rcS        boot sequence
/tmp/router/etc/passwd            login (root, no password)
/tmp/router/web/language_en.js    3400 UI strings
/tmp/router/etc/conf/rtl8852ce/   Wi-Fi calibration data
```

### Flash

```bash
./flash.sh --tty /dev/ttyUSB0 --image-dir ~/sdk/.../target/image
```

Requires a USB-serial adapter and a physical TL-WR1502X v1.
See [docs/FLASHING.md](docs/FLASHING.md) for step-by-step.

---

## Files this build produces

| File | Size | Purpose |
|---|---|---|
| `fw.bin` | 7.58 MB | Full firmware — flash this |
| `linux.bin` | 1.99 MB | Kernel only (TFTP recovery) |
| `root.bin` | 5.60 MB | Squashfs rootfs |
| `nfjrom.bin` | 1.99 MB | Bootloader programmer image |
| `vmlinux.elf` | 7.32 MB | Unstripped kernel (debug) |
| `config.dat` | 10 KB | Default settings blob |

---

## What this repository contains

- **`build.sh`** — end-to-end build script with patch application,
  environment checks, and artifact verification
- **`extract-rootfs.sh`** — extracts the squashfs from `root.bin`
  (deals with the 16-byte Realtek header)
- **`flash.sh`** — automates serial + TFTP flashing
- **`patches/patched-tree.tar.bz2`** — all source modifications needed
  for GCC 15 / modern glibc

## What this repository does NOT contain

- The original GPL source (download from TP-Link separately)
- The MIPS cross-toolchain (~200 MB, in the SDK tarball)
- Any TP-Link proprietary binaries not covered by GPL

---

## The patches — what they fix

Every change in `patched-tree.tar.bz2` addresses a specific
GCC-15-vs-2008-code incompatibility. Full breakdown:

### Shell / build system

| Issue | Fix |
|---|---|
| `[[: not found` under dash | `config/*` shebangs → `#!/bin/bash` |
| `CFLAGS` references itself | Removed trailing `$(CFLAGS)` from ~15 Makefiles |
| Dashed config symbols break `. config` | Filter `users/.config` before sourcing |
| Unquoted `[ -n $VAR ]` | Quoted everywhere |

### Compiler errors (GCC 14+ strictness)

| Issue | Fix location |
|---|---|
| `typedef enum {false, true} bool` vs C23 | `users/*/apmib/list.h` C23 guard |
| Implicit function declarations | ~250 Makefiles `-Wno-error=implicit-function-declaration` |
| K&R function definitions | `-Wno-error=implicit-int` |
| Int-to-pointer casts | `-Wno-error=int-to-pointer-cast` |
| Incompatible pointer types | `-Wno-error=incompatible-pointer-types` |

**Do not add** `-Wno-error=return-mismatch` or
`-Wno-error=declaration-missing-parameter-type` — the MIPS cross-toolchain
(GCC 4.9.4) doesn't recognize them and errors out.

### Environment

| Issue | Fix |
|---|---|
| `qemu-mipsel` binfmt hangs MIPS binary probes | `build.sh` disables it |
| Menuconfig picks wrong toolchain (3.10) | `build.sh` verifies `.config` |

---

## Register bases (authoritative, from `bspchip.h`)

| Peripheral | Base | IRQ | Clock |
|---|---|---|---|
| UART0 | `0x18147000` | ICTL0 + 9 | 100 MHz |
| UART1 | `0x18147400` | ICTL1 + 6 | 100 MHz |
| UART2 | `0x18147800` | ICTL1 + 7 | 100 MHz |
| Timer0 | `0xBF201000` | CPU + 6 | 25 MHz |
| TC | `0xB8003100` | ICTL0 + 7 | — |
| GPIO | `0xB8003500` | ICTL0 + 16/17 | — |
| SPI flash | `0x18143000` | — | 200 MHz |
| PCIe RC | `0xB8B00000` | CPU + 5 | — |
| USB OTG | `0x18100000` | ICTL0 + 12 | — |
| I2C0/1 | `0x1801D000/100` | ICTL0 + 30 / ICTL1 + 3 | 100 MHz |
| SSI0/1 | `0x1801C000/100` | ICTL0 + 23 / ICTL1 + 5 | 100 MHz |

IRQ space: CPU 0–7, ICTL0 8–39, ICTL1 40–71, GPIO 72+.

---

## Troubleshooting

### Build hangs with no output

`qemu-mipsel` binfmt is intercepting MIPS binary probes. The build
script disables it automatically, but if you see a hang:

```bash
pkill -9 qemu-mipsel
echo 0 | sudo tee /proc/sys/fs/binfmt_misc/qemu-mipsel
```

### `ERROR: unknown kernel configuration`

Menuconfig didn't write `CONFIG_LINUX_4.4=y`. Re-run with:

```bash
cd rtl8197 && make SHELL=/bin/bash menuconfig
```

And confirm `.config` has `CONFIG_LINUX_4.4=y` afterward.

### `cc1: error: -Werror=return-mismatch: no option`

A GCC-14-only flag leaked into the cross build. Remove:

```bash
grep -rl 'return-mismatch' rtl8197/users/ --include=Makefile | \
  xargs sed -i 's|-Wno-error=return-mismatch ||g'
```

### `mkimg` fails sourcing `users/.config`

The `patched-tree.tar.bz2` already fixes this. If it recurs:

```bash
grep -n 'users/.config' rtl8197/target/tools/mkimg
```

Verify the source line uses the filtered version, not raw `.`.

### Router doesn't boot after flashing

See [docs/FLASHING.md](docs/FLASHING.md#troubleshooting). Most common:

1. **No serial output** — wrong baud. This board is 38400, not 115200.
2. **Bootloader rejects image** — signature check; use `nfjrom.bin`
   via programmer mode or stock TP-Link recovery.
3. **Kernel boots, no rootfs** — wrong load address; repackage with
   `cvimg` using `0x80A00000 0x900000`.

---

## Contributing

Patches welcome for:

- Testing on real hardware (issues + boot logs)
- Adding bootloader signature bypass documentation
- Porting to OpenWrt (separate project)
- Reproducing with GCC 12/13/14
- Cleaning up the GCC 15 workarounds into proper source fixes

Open an issue with:
- Host OS and GCC version
- Full `build.log` on failure
- Serial boot log if hardware-related

---

## License

- **Patches and scripts in this repository:** GPL-2.0
- **Original SDK source:** © Realtek Semiconductor Corp. and TP-Link,
  licensed under GPL-2.0 as published by TP-Link

This repository does not distribute the original SDK, only the
patches needed to make it build on modern systems.

---

## Acknowledgements

- Realtek Semiconductor — original SDK
- TP-Link — GPL source release
- The MIPS and OpenWrt communities — reference material

## Disclaimer

Flashing custom firmware voids your warranty and can brick your
device. This build is **untested on physical hardware**. If you flash
it, you accept the risk. Keep a copy of your factory firmware and a
serial connection.
ENDREADME
```

## `build.sh`

```bash
cat > build.sh <<'ENDSCRIPT'
#!/usr/bin/env bash
# TL-WR1502X v1 firmware build script
#
# Usage:
#   ./build.sh --sdk <path> [--apply-patches <path>] [--clean] [--verify]
#
# Examples:
#   # Full build from clean SDK (recommended)
#   ./build.sh --sdk ~/sdk/GPL_TL-WR1502Xv1 --apply-patches patches/patched-tree.tar.bz2
#
#   # Rebuild after making source changes
#   ./build.sh --sdk ~/sdk/GPL_TL-WR1502Xv1
#
#   # Just verify existing artifacts
#   ./build.sh --sdk ~/sdk/GPL_TL-WR1502Xv1 --verify

set -euo pipefail

# -------- defaults --------
SDK_ROOT=""
PATCH_FILE=""
DO_CLEAN=0
DO_VERIFY=0
TOOLCHAIN="msdk-6.4.1-mips-EL-4.4-u0.9.33-m32ut-190619"
BOARD="SPINAND_RTL8197F_VG_8832BR_8367R_KERN44_GW"
FW_MD5_KNOWN="a22b083a7f2e4f4667f0b9b5167a614d"

# -------- colors --------
if [ -t 1 ]; then
    B=$'\033[1;34m'; G=$'\033[1;32m'; Y=$'\033[1;33m'
    R=$'\033[1;31m'; N=$'\033[0m'
else
    B=""; G=""; Y=""; R=""; N=""
fi

log()  { printf '%s[build]%s %s\n' "$B" "$N" "$*"; }
ok()   { printf '%s[  ok]%s %s\n' "$G" "$N" "$*"; }
warn() { printf '%s[warn]%s %s\n' "$Y" "$N" "$*"; }
err()  { printf '%s[err ]%s %s\n' "$R" "$N" "$*" >&2; }
die()  { err "$*"; exit 1; }

usage() {
    sed -n '2,20p' "$0" | sed 's/^# \?//'
    exit 1
}

# -------- args --------
while [ $# -gt 0 ]; do
    case "$1" in
        --sdk)            SDK_ROOT="$2"; shift 2 ;;
        --apply-patches)  PATCH_FILE="$2"; shift 2 ;;
        --clean)          DO_CLEAN=1; shift ;;
        --verify)         DO_VERIFY=1; shift ;;
        -h|--help)        usage ;;
        *) die "unknown arg: $1" ;;
    esac
done

[ -n "$SDK_ROOT" ] || { err "--sdk <path> is required"; usage; }
[ -d "$SDK_ROOT/rtl8197" ] || die "SDK not found at $SDK_ROOT/rtl8197"

# -------- preflight --------
preflight() {
    log "Preflight checks..."

    # Host tools
    for t in make gcc g++ bison flex; do
        command -v "$t" >/dev/null || die "missing tool: $t"
    done
    ok "host toolchain present"

    # Cross toolchain
    local tc="$SDK_ROOT/rtl8197/toolchain/$TOOLCHAIN/bin"
    [ -x "$tc/mips-linux-uclibc-gcc" ] \
        || die "cross toolchain missing at $tc"
    ok "MIPS cross-toolchain found"

    # qemu-mipsel binfmt (disables MIPS binary probes)
    if [ -f /proc/sys/fs/binfmt_misc/qemu-mipsel ]; then
        local state
        state=$(head -1 /proc/sys/fs/binfmt_misc/qemu-mipsel 2>/dev/null || true)
        if [ "$state" = "enabled" ]; then
            warn "disabling qemu-mipsel binfmt (hangs MIPS probes)"
            echo 0 | sudo tee /proc/sys/fs/binfmt_misc/qemu-mipsel >/dev/null \
                || warn "could not disable; build may hang"
        fi
    fi

    # /bin/sh is bash? (config/* scripts need it)
    if ! sh -c '[[ 1 == 1 ]]' 2>/dev/null; then
        warn "/bin/sh is not bash; patching config/* shebangs"
        for f in "$SDK_ROOT"/rtl8197/config/*; do
            [ -f "$f" ] || continue
            head -1 "$f" | grep -q '^#! */bin/sh' \
                && sed -i '1s|^#! */bin/sh|#!/bin/bash|' "$f"
        done
        ok "shebangs patched"
    fi
}

# -------- patches --------
apply_patches() {
    [ -n "$PATCH_FILE" ] || return 0
    [ -f "$PATCH_FILE" ] || die "patch file not found: $PATCH_FILE"
    log "Applying patches from $PATCH_FILE"
    tar -xjf "$PATCH_FILE" -C "$(dirname "$SDK_ROOT")" \
        || die "patch extraction failed"
    ok "patches applied"
}

# -------- clean --------
do_clean() {
    [ "$DO_CLEAN" = 1 ] || return 0
    log "Cleaning previous build..."
    cd "$SDK_ROOT/rtl8197"
    rm -rf target/{image,romfs,tmpfs,bsp} 2>/dev/null || true
    rm -f linux-4.4/vmlinux linux-4.4/rtkload/linux.bin 2>/dev/null || true
    find . -name '*.o' -newer Makefile -delete 2>/dev/null || true
    ok "clean done"
}

# -------- config --------
configure() {
    log "Configuration check..."
    cd "$SDK_ROOT/rtl8197"

    if [ ! -f .config ]; then
        warn ".config missing — running menuconfig (interactive)"
        if [ -t 0 ]; then
            make SHELL=/bin/bash menuconfig
        else
            die "menuconfig needs a TTY; run interactively or provide .config"
        fi
    fi

    # Verify toolchain
    if ! grep -q "^CONFIG_RSDK_${TOOLCHAIN}=y" .config; then
        warn "toolchain defaulted to wrong version — patching .config"
        sed -i "s|^CONFIG_RSDK_msdk-4.4.7.*|# &|" .config
        sed -i "s|^# CONFIG_RSDK_${TOOLCHAIN} is not set|CONFIG_RSDK_${TOOLCHAIN}=y|" .config
        sed -i "s|^CONFIG_RSDKDIR=.*|CONFIG_RSDKDIR=toolchain/${TOOLCHAIN}|" .config
    fi
    ok "config ready"
}

# -------- build --------
build() {
    log "Building (this takes 30–90 minutes)..."
    cd "$SDK_ROOT/rtl8197"
    local logf="$SDK_ROOT/build.log"

    if ! make 2>&1 | tee "$logf"; then
        err "build failed — last 40 lines:"
        tail -40 "$logf" >&2
        die "see $logf"
    fi
    ok "build complete"
}

# -------- verify --------
verify() {
    log "Verifying artifacts..."
    local img="$SDK_ROOT/rtl8197/target/image"
    [ -f "$img/fw.bin" ] || die "fw.bin not produced"

    cd "$img"
    md5sum fw.bin linux.bin root.bin nfjrom.bin vmlinux.elf 2>/dev/null

    local md5
    md5=$(md5sum fw.bin | awk '{print $1}')
    if [ "$md5" = "$FW_MD5_KNOWN" ]; then
        ok "fw.bin matches reference ($FW_MD5_KNOWN)"
    else
        warn "fw.bin md5 differs from reference:"
        warn "  expected: $FW_MD5_KNOWN"
        warn "  got:      $md5"
        warn "  (only an issue if unmodified source produced this)"
    fi
}

# -------- main --------
main() {
    preflight

    if [ "$DO_VERIFY" = 1 ]; then
        verify
        exit 0
    fi

    apply_patches
    do_clean
    configure
    build
    verify

    log ""
    ok "Done."
    log "Firmware: $SDK_ROOT/rtl8197/target/image/fw.bin"
    log "Flash:    ./flash.sh --image-dir $SDK_ROOT/rtl8197/target/image"
    log "Inspect:  ./extract-rootfs.sh $SDK_ROOT/rtl8197/target/image/root.bin /tmp/router"
}

main "$@"
ENDSCRIPT

chmod +x build.sh
```

## `extract-rootfs.sh`

```bash
cat > extract-rootfs.sh <<'ENDSCRIPT'
#!/usr/bin/env bash
# Extract the squashfs from a Realtek root.bin
#
# Usage:
#   ./extract-rootfs.sh <root.bin> <output-dir>
#
# The SDK wraps the squashfs in a 16-byte Realtek header. This script
# finds the squashfs magic (hsqs) and extracts from that offset.

set -euo pipefail

if [ $# -ne 2 ]; then
    sed -n '2,10p' "$0" | sed 's/^# \?//'
    exit 1
fi

IMG="$1"
OUT="$2"

[ -f "$IMG" ] || { echo "not found: $IMG" >&2; exit 1; }

# find first hsqs magic
OFFSET=$(grep -abo 'hsqs' "$IMG" | head -1 | cut -d: -f1)

if [ -z "$OFFSET" ]; then
    echo "no squashfs magic found in $IMG" >&2
    echo "Is this really a Realtek root.bin?" >&2
    exit 1
fi

echo "squashfs starts at offset $OFFSET"
echo "extracting to $OUT ..."

mkdir -p "$OUT"
unsquashfs -offset "$OFFSET" -d "$OUT" "$IMG"

echo ""
echo "Done. Interesting files:"
echo "  $OUT/etc/init.d/rcS        boot sequence"
echo "  $OUT/etc/passwd            login accounts"
echo "  $OUT/web/language_en.js    UI strings"
echo "  $OUT/etc/conf/rtl8852ce/   Wi-Fi calibration"
echo "  $OUT/bin/                  userspace binaries"
ENDSCRIPT

chmod +x extract-rootfs.sh
```

## `flash.sh`

```bash
cat > flash.sh <<'ENDSCRIPT'
#!/usr/bin/env bash
# TFTP-flash helper for TL-WR1502X v1
#
# Requires:
#   - Physical router connected via serial (USB-serial adapter)
#   - Ethernet cable to a LAN port
#   - Router in bootloader prompt (<RealTek>) via ESC during boot
#
# Usage:
#   ./flash.sh --tty /dev/ttyUSB0 --image-dir <path> [--host 192.168.1.6]
#
# Notes:
#   - Router defaults to 192.168.1.6 as TFTP client
#   - Your PC must be at 192.168.1.1/24 on the LAN side
#   - Serial console: 38400 8N1, no flow control

set -euo pipefail

TTY=""
IMAGE_DIR=""
ROUTER_IP="192.168.1.6"

usage() {
    sed -n '2,15p' "$0" | sed 's/^# \?//'
    exit 1
}

while [ $# -gt 0 ]; do
    case "$1" in
        --tty)        TTY="$2"; shift 2 ;;
        --image-dir)  IMAGE_DIR="$2"; shift 2 ;;
        --host)       ROUTER_IP="$2"; shift 2 ;;
        -h|--help)    usage ;;
        *) echo "unknown arg: $1" >&2; usage ;;
    esac
done

[ -n "$TTY" ] || { echo "--tty is required" >&2; usage; }
[ -n "$IMAGE_DIR" ] || { echo "--image-dir is required" >&2; usage; }
[ -d "$IMAGE_DIR" ] || { echo "not a directory: $IMAGE_DIR" >&2; exit 1; }

for f in linux.bin root.bin; do
    [ -f "$IMAGE_DIR/$f" ] || { echo "missing $IMAGE_DIR/$f" >&2; exit 1; }
done

command -v tftp >/dev/null || { echo "install tftp: sudo apt install tftp" >&2; exit 1; }

cat <<EOF

==============================================================
  TL-WR1502X v1 flashing
==============================================================

  Serial:      $TTY @ 38400 8N1, no flow control
  Image dir:   $IMAGE_DIR
  Router IP:   $ROUTER_IP (TFTP client)

  Before running:
    1. Ensure PC is at 192.168.1.1/24 on the LAN side
    2. Power on the router
    3. Press ESC during boot to reach the <RealTek> prompt
    4. Then come back here

  Press Enter to continue, Ctrl-C to abort.
==============================================================
EOF
read -r

echo ""
echo "Uploading linux.bin ($(stat -c%s "$IMAGE_DIR/linux.bin") bytes)..."
cd "$IMAGE_DIR"
tftp "$ROUTER_IP" <<TFTP
binary
put linux.bin
put root.bin
quit
TFTP

echo ""
echo "Upload done. Router should auto-boot."
echo "Watch the serial console for:"
echo "  'Linux version 4.4.176 ...'"
ENDSCRIPT

chmod +x flash.sh
```

## `.gitignore`

```bash
cat > .gitignore <<'END'
# Build outputs
*.log
build/
target/
vmlinux
vmlinux.elf
fw.bin
linux.bin
root.bin
nfjrom.bin
config.dat

# Editor
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db
END
```

## `LICENSE`

```bash
cat > LICENSE <<'END'
Patches and scripts in this repository are licensed under GPL-2.0.

The original SDK source (not included here) is copyright
Realtek Semiconductor Corp. and TP-Link, licensed under GPL-2.0
as published in TP-Link's GPL source release.

See https://www.gnu.org/licenses/old-licenses/gpl-2.0.html
END
```

## Final structure

```bash
ls -la ~/tl-wr1502x-firmware/
ls -la ~/tl-wr1502x-firmware/patches/
```

Expected:

```
tl-wr1502x-firmware/
├── .gitignore
├── LICENSE
├── README.md
├── build.sh                 (executable)
├── extract-rootfs.sh        (executable)
├── flash.sh                 (executable)
└── patches/
    └── patched-tree.tar.bz2
```

## Test the scripts

```bash
cd ~/tl-wr1502x-firmware

# Help text works
./build.sh --help
./extract-rootfs.sh
./flash.sh --help

# Extract script works on existing root.bin
./extract-rootfs.sh ~/tl-wr1502x-build/20261006/root.bin /tmp/router-extract
ls /tmp/router-extract/
```

## Push to GitHub

```bash
cd ~/tl-wr1502x-firmware

git init
git add .
git commit -m "Initial commit: TL-WR1502X v1 GPL firmware rebuild"

# Create repo on github.com, then:
git remote add origin git@github.com:<youruser>/tl-wr1502x-firmware.git
git branch -M main
git push -u origin main
```

## What people will see

On GitHub, the README renders with:

- Hardware table
- Build status badges
- Quick-start commands in copy-paste blocks
- Troubleshooting section
- Register bases table (useful for anyone porting to OpenWrt)
- Clear "not tested on hardware" disclaimer

Anyone with the same router can clone your repo, download TP-Link's GPL tarball, and produce the same `fw.bin` in one command. That's the whole point of publishing it.

**Two things to check before pushing:**

1. **Does TP-Link's GPL license allow redistribution of the patches?** — Yes, GPL requires it. Your patches are derivative works of GPL code, so they must be GPL too. That's why `LICENSE` says GPL-2.0.

2. **Is `patched-tree.tar.bz2` too big for GitHub?** — At 98 KB, no. But if you want to make patches more readable, you could expand it into individual `.patch` files. For now, the tarball is fine.
