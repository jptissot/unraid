# Unraid 7.3 Kernel Rebuild — AMD GPU P2P Enablement

Rebuilds the Unraid kernel with peer-to-peer DMA enabled between AMD GPUs
(dual Radeon AI PRO R9700, gfx1201). Verified against Unraid 7.3.2
(kernel 6.18.38). Version-agnostic where possible: kernel and ZFS versions
are detected from the running system.

**Flags enabled:**

```
CONFIG_PCI_P2PDMA=y
CONFIG_DMABUF_MOVE_NOTIFY=y
CONFIG_HSA_AMD_P2P=y
```

## How Unraid 7.3 boot files actually work (learned the hard way)

The legacy filenames on `/boot` no longer mean what they say:

| File | Actual contents (7.3) | Touched by this guide |
|---|---|---|
| `bzimage` | Kernel image (md/unraid driver **built in**) | **Replaced** |
| `bzroot` | Initramfs: uncompressed early cpio (CPU microcode) + zstd-compressed main cpio containing the root fs **and `/lib/modules`** | **Replaced** (module tree swapped) |
| `bzmodules` | `/usr` squashfs — **NOT kernel modules** | Never touched |
| `bzfirmware` | `/lib/firmware` squashfs | Never touched |

Replacing `bzmodules` with a modules squashfs (the pre-7.3 procedure)
**bricks the boot** — the system would come up without `/usr`.

Other facts this guide relies on:

- Unraid ships a build recipe at `/usr/src/linux-$(uname -r)/`: stock
  `config`, patches, and a `drivers/` overlay containing the md/unraid
  array driver source. Vanilla kernel.org source + recipe = stock kernel.
- `CONFIG_MD_UNRAID` is `bool` — the array driver compiles **into
  bzimage**, never as a module.
- ZFS is out-of-tree (not in the kernel source); it must be built
  separately against the new tree, version-matched to the userspace tools
  in bzroot.
- Unraid verifies `*.sha256` checksums on `/boot` at boot.
- **Every Unraid OS update overwrites these files. Redo this entire
  procedure after every update.**

## Prerequisites (one-time, BIOS)

Above 4G Decoding and Resizable BAR ON. Verify full-VRAM BARs:

```bash
lspci -vvv -d 1002: | grep -A1 "Region 0"    # expect [size=32G] on each R9700
```

---

## Phase 0 — Gather versions and clean the workspace (HOST)

```bash
KVER=$(uname -r)                      # e.g. 6.18.38-Unraid
ZVER=$(cat /sys/module/zfs/version)   # e.g. 2.4.3-1 -> upstream is 2.4.3
echo "kernel: $KVER   zfs: $ZVER"

# clean workspace (removes any previous build)
rm -rf /mnt/user/appdata/kernel-build
mkdir -p /mnt/user/appdata/kernel-build

# stage a copy of bzroot for the container (it cannot see /boot)
cp /boot/bzroot /mnt/user/appdata/kernel-build/
```

Strip the package revision from ZVER by hand: `2.4.3-1` → use `2.4.3`
in Phase 3.

---

## Phase 1 — Build container (HOST → CONTAINER)

```bash
docker run -it --rm \
  -v /mnt/user/appdata/kernel-build:/build \
  -v /usr/src/linux-$(uname -r):/unraid-src:ro \
  -e KVER=$(uname -r) \
  -w /build ubuntu:24.04
```

Everything below runs **inside this one container session**. If you ever
restart the container, re-run the `apt install` line — tools do not
persist, and a missing tool in a pipeline can silently produce empty
output (this bit us: `zstd` missing produced a truncated bzroot that the
integrity check in Phase 5 caught).

```bash
set -o pipefail    # make pipeline failures loud

apt update && apt install -y build-essential bc bison flex libssl-dev \
  libelf-dev cpio kmod squashfs-tools wget xz-utils zstd file \
  initramfs-tools-core \
  autoconf automake libtool gawk uuid-dev libblkid-dev \
  libudev-dev libaio-dev libattr1-dev zlib1g-dev python3
```

---

## Phase 2 — Kernel build (CONTAINER)

### 2.1 Fetch matching vanilla source

```bash
KSRC=${KVER%-Unraid}
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-${KSRC}.tar.xz
tar xf linux-${KSRC}.tar.xz
cd linux-${KSRC}
```

### 2.2 Apply Unraid patches

```bash
for p in /unraid-src/*.patch; do
  echo "== $p"
  patch -p1 --no-backup-if-mismatch < "$p" || { echo "FAILED: $p"; break; }
done
```

**CHECKPOINT:** every patch reports `patching file ...`. Offsets and
fuzz are fine; any `FAILED` or `.rej` is a stop.

### 2.3 Overlay the array driver source

```bash
cp -rv /unraid-src/drivers/* drivers/
```

### 2.4 Config: stock + P2P flags

```bash
cp /unraid-src/config .config
./scripts/config --enable PCI_P2PDMA \
                 --enable DMABUF_MOVE_NOTIFY \
                 --enable HSA_AMD_P2P
make olddefconfig
```

**CHECKPOINT** (olddefconfig silently drops flags with unmet deps):

```bash
grep -E 'PCI_P2PDMA=|DMABUF_MOVE_NOTIFY=|HSA_AMD_P2P=|CONFIG_LOCALVERSION=|MD_UNRAID=' .config
```

Required, exactly:

```
CONFIG_LOCALVERSION="-Unraid"
CONFIG_MD_UNRAID=y
CONFIG_PCI_P2PDMA=y
CONFIG_HSA_AMD_P2P=y
CONFIG_DMABUF_MOVE_NOTIFY=y
```

### 2.5 Compile (10–30 min)

```bash
make -j$(nproc) bzImage modules
```

**CHECKPOINT** — array driver compiled into the image (it is a
built-in, so it will never appear as a `.ko`):

```bash
ls -la drivers/md/md_unraid.o drivers/md/unraid.o   # fresh timestamps
```

### 2.6 Stage modules

```bash
mkdir -p /build/staging && rm -rf /build/staging/lib
make INSTALL_MOD_PATH=/build/staging modules_install
```

---

## Phase 3 — ZFS build (CONTAINER)

Must run **after** Phase 2 (configure inspects the built tree). Use the
upstream version matching the host's `/sys/module/zfs/version` minus the
package revision.

```bash
cd /build
ZVER=2.4.3    # <-- EDIT to match Phase 0
wget https://github.com/openzfs/zfs/releases/download/zfs-${ZVER}/zfs-${ZVER}.tar.gz
tar xf zfs-${ZVER}.tar.gz && cd zfs-${ZVER}
./configure --with-config=kernel \
  --with-linux=/build/linux-${KVER%-Unraid} \
  --with-linux-obj=/build/linux-${KVER%-Unraid}
make -j$(nproc)
make DESTDIR=/build/staging install
```

**CHECKPOINT:**

```bash
find /build/staging/lib/modules/${KVER} -name 'zfs.ko*' -o -name 'spl.ko*'
# both must appear (in extra/, .xz-compressed)
```

---

## Phase 4 — Module accounting (HOST, container stays open)

Ground truth is the **live** `/lib/modules` (what the system actually
boots with), not any mounted archive.

```bash
find /lib/modules/$(uname -r) -name '*.ko*' -printf '%f\n' | sort -u > /tmp/runtime-modules.txt
find /mnt/user/appdata/kernel-build/staging/lib/modules -name '*.ko*' -printf '%f\n' | sort -u > /tmp/new-modules.txt

wc -l /tmp/runtime-modules.txt /tmp/new-modules.txt   # both must be ~1100+; a tiny count = a find failed
comm -23 /tmp/runtime-modules.txt /tmp/new-modules.txt   # modules we'd LOSE — must be empty
```

**CHECKPOINT:** the `comm` output is empty. Anything listed is a module
the running system has that the new build lacks — resolve before
continuing (out-of-tree component or plugin-injected module; build it
against the new tree like ZFS, do not copy prebuilt `.ko`s, because
`PCI_P2PDMA` changes `struct pci_dev` layout).

---

## Phase 5 — bzroot surgery (CONTAINER)

### 5.1 Extract

```bash
cd /build
mkdir -p bzroot-extract
unmkinitramfs /build/bzroot /build/bzroot-extract
ls bzroot-extract                          # expect: early  main
find bzroot-extract -name '*.ko*' | wc -l  # expect same count as runtime list
```

### 5.2 Swap in the new module tree + metadata

```bash
rm -rf /build/bzroot-extract/main/lib/modules/${KVER}
cp -a /build/staging/lib/modules/${KVER} /build/bzroot-extract/main/lib/modules/

# metadata that modules_install leaves in the source tree; without
# modules.builtin, modprobe misreports built-in modules as missing
cp /build/linux-${KVER%-Unraid}/modules.order \
   /build/linux-${KVER%-Unraid}/modules.builtin \
   /build/linux-${KVER%-Unraid}/modules.builtin.modinfo \
   /build/bzroot-extract/main/lib/modules/${KVER}/

depmod -b /build/bzroot-extract/main ${KVER}
```

**CHECKPOINT:** depmod prints **zero warnings**.

### 5.3 Repack (early = plain cpio, main = zstd cpio)

```bash
cd /build/bzroot-extract/early
find . | sort | cpio -o -H newc --owner root:root > /build/early.cpio

cd /build/bzroot-extract/main
find . | sort | cpio -o -H newc --owner root:root | zstd -19 -T0 > /build/main.img

ls -la /build/early.cpio /build/main.img
# early ~17 MB, main ~60-70 MB. A zero/tiny main.img means a pipeline
# tool was missing — fix and re-run.

cat /build/early.cpio /build/main.img > /build/bzroot-new
cp /build/linux-${KVER%-Unraid}/arch/x86/boot/bzImage /build/bzimage-new
```

### 5.4 Integrity gate — do not skip

```bash
EARLY=$(stat -c%s /build/early.cpio)
head -c $EARLY /build/bzroot-new | cmp - /build/early.cpio && echo "early OK"
tail -c +$((EARLY+1)) /build/bzroot-new | zstd -t && echo "main OK"
ls -la /build/bzroot /build/bzroot-new    # same ballpark size (~82 MB)
```

**CHECKPOINT:** both `OK` lines print and sizes are comparable. Only
then `exit` the container.

---

## Phase 6 — Install (HOST)

Only `bzimage` and `bzroot` change. **Never** touch `bzmodules` or
`bzfirmware`.

```bash
ls -la /mnt/user/appdata/kernel-build/bzimage-new /mnt/user/appdata/kernel-build/bzroot-new
# bzimage-new ~9-10 MB, bzroot-new ~82 MB — confirm both exist first

cp /boot/bzimage /boot/bzimage.stock
cp /boot/bzroot  /boot/bzroot.stock

cp /mnt/user/appdata/kernel-build/bzimage-new /boot/bzimage
cp /mnt/user/appdata/kernel-build/bzroot-new  /boot/bzroot

cd /boot
sha256sum bzimage > bzimage.sha256
sha256sum bzroot  > bzroot.sha256
sha256sum -c bzimage.sha256 bzroot.sha256     # both OK
ls -la /boot/bzimage.stock /boot/bzroot.stock # backups present
```

Reboot cleanly from the web UI (or `powerdown -r`).

---

## Phase 7 — Verify

No boot/cmdline flags are needed for P2P — the feature is compile-time
plus BIOS large-BAR; `amdgpu.pcie_p2p` defaults to on (setting it to 0
on the cmdline is the kill switch if P2P ever needs disabling without a
rebuild).

In order — each step gates the next:

```bash
uname -r                                   # matches the version you built
cat /proc/version                          # your build date + gcc 13.x (Ubuntu
                                           # toolchain), not Limetech's build
# array starts in the web UI              -> md/unraid built-in works
zpool status                               # pools imported -> ZFS build works
                                           # (do NOT 'zpool upgrade' — keeps rollback safe)

# P2P flag compiled into the RUNNING kernel (there is no /proc/config.gz —
# stock config lacks IKCONFIG, so grep built-in symbols instead):
grep -c p2pdma /proc/kallsyms              # >0 (dozens) = PCI_P2PDMA is in

# amdgpu module compiled with HSA_AMD_P2P (file only exists if it was):
cat /sys/module/amdgpu/parameters/pcie_p2p # Y = feature present and enabled
```

**The decisive kernel-side check** — KFD peer links. These live in
`p2p_links/`, NOT `io_links/` (io_links only holds CPU<->GPU links from
firmware topology; the links `HSA_AMD_P2P` creates are separate):

```bash
for n in /sys/class/kfd/kfd/topology/nodes/*/; do
  echo "== node ${n}"
  grep -E 'p2p_links_count|gfx_target' $n/properties 2>/dev/null
  for l in $n/p2p_links/*/; do
    [ -d "$l" ] && grep -E '^(type|node_from|node_to)' $l/properties | tr '\n' ' ' && echo
  done
done
```

**PASS:** each R9700 node (`gfx_target_version 120001`) has a
`type 2` (PCIe) p2p_link whose `node_to` is the other R9700's node
number. The iGPU (gfx_target_version 1003xx) getting links too is
normal and harmless — keep it excluded from ROCm containers via
`HIP_VISIBLE_DEVICES`/`ROCR_VISIBLE_DEVICES`.

**FAIL** (no GPU<->GPU p2p_links): check
`dmesg | grep -i amdgpu | grep -i chipset` for the "P2P access ... not
supported by the chipset" message (the only loudly-logged failure), and
verify large BAR as amdgpu sees it — on each dGPU,
`mem_info_vram_total` must equal `mem_info_vis_vram_total` under
`/sys/class/drm/card*/device/`.

Then from a ROCm container (with `/dev/kfd` and `/dev/dri` mapped —
ROCm tools run in containers, not on the Unraid host):

```bash
rocm-smi --showtopo
rocm_bandwidth_test    # peer-to-peer section: real GB/s, not N/A
```

Finally: remove `NCCL_P2P_DISABLE=1` from the vLLM container, restart,
and validate with rccl-tests before real workloads:

```bash
all_reduce_perf -b 8 -e 256M -g 2
```

> **RCCL heads-up:** known RCCL deadlock on dual R9700 (gfx1201) TP=2
> over PCIe introduced between RCCL 2.27.3 and 2.27.7
> (ROCm/rocm-systems #5480) — both GPUs spin at 100% on the first
> multi-GPU op. A hang at graph capture or first forward pass is more
> likely an RCCL version issue than this kernel build.

---

## Rollback (fails to boot)

1. Power off; move the Unraid flash drive to another computer.
2. Rename `bzimage.stock` → `bzimage`, `bzroot.stock` → `bzroot`.
3. Delete `bzimage.sha256` and `bzroot.sha256` (or regenerate on a
   Linux box) so the boot checksum verification matches.
4. Boot stock; debug from the build artifacts in
   `/mnt/user/appdata/kernel-build`.

## Notes

- Redo everything after each Unraid update. Versions are auto-detected;
  only the `ZVER=` line in Phase 3 needs a manual edit if Unraid bumps
  ZFS.
- If Unraid moves to a 7.x Linux kernel, change `v6.x` in the Phase 2
  URL to `v7.x` and re-verify the recipe patches apply.
- The stock `bzroot`'s compression was verified as zstd on 7.3. If a
  future release changes it, detect it by scanning the copy for magic
  bytes (`xz \xfd7zXZ\x00`, `zstd \x28\xb5\x2f\xfd`, `gzip \x1f\x8b\x08`)
  — the **lowest offset** is the real main stream; later hits are noise
  inside compressed data. For an xz repack, `xz --check=crc32` is
  mandatory (kernel rejects CRC64).
- Config-only changes normally preserve module ABI, but `PCI_P2PDMA`
  modifies `struct pci_dev` — rebuild any out-of-tree modules from
  source rather than copying prebuilt ones.