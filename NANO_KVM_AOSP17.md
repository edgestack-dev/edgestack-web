# Android 17 (Cuttlefish ARM64) on NVIDIA Orin Nano via crosvm/KVM

> Delta guide for booting Android 17 in a KVM VM on Orin Nano (L4T/Ubuntu 22.04).
> Base reference: [RPI5_KVM_AOSP17_CF.md](RPI5_KVM_AOSP17_CF.md) — follow that doc first, then apply the deltas below.

---

## Table of Contents

1. [Host Differences (Orin vs RPi5)](#1-host-differences-orin-vs-rpi5)
2. [Additional Dependencies](#2-additional-dependencies)
3. [Build virglrenderer from Source](#3-build-virglrenderer-from-source)
4. [Build crosvm](#4-build-crosvm)
5. [Patch vbmeta (Disable AVB Hashtree Verification)](#5-patch-vbmeta-disable-avb-hashtree-verification)
6. [Patch Vendor Image (Disable Encryption in super.img)](#6-patch-vendor-image-disable-encryption-in-superimg)
7. [Prepare Metadata and Userdata](#7-prepare-metadata-and-userdata)
8. [Boot Command (Orin Nano)](#8-boot-command-orin-nano)
9. [ADB Access](#9-adb-access)
10. [Display Output on DP](#10-display-output-on-dp)
11. [Troubleshooting (Orin-Specific)](#11-troubleshooting-orin-specific)
12. [Boot via run_cvd (Full Cuttlefish Infrastructure)](#12-boot-via-run_cvd-full-cuttlefish-infrastructure)
13. [Physical Display + Touch (Weston Fullscreen)](#13-physical-display--touch-weston-fullscreen)

---

## 1. Host Differences (Orin vs RPi5)

| Aspect | RPi5 | Orin Nano |
|--------|-------|-----------|
| OS | Debian Bookworm / Ubuntu 24.04 | L4T Ubuntu 22.04 (JetPack) |
| Kernel | 6.x mainline | 5.15.148-tegra |
| RAM | 4GB (1GB to VM) | 8GB (2GB to VM) |
| `vhost_vsock` | Available via `modprobe` | Must build from L4T kernel source (see Section 9) |
| virglrenderer | System package (>=1.0) | Must build from source (L4T ships 0.9.1) |
| `libminijail-dev` | System package | **Not available** — crosvm builds from submodule |
| ADB transport | vsock (`--cid 3`) | vsock (`--cid 3`) — after building vsock modules |

---

## 2. Additional Dependencies

Beyond what the RPi5 doc lists, install:

```bash
sudo apt install -y \
    clang \
    libclang-dev \
    libepoxy-dev \
    meson \
    ninja-build \
    curl
```

- `clang` / `libclang-dev` — required by crosvm's `bindgen` for FFI generation
- `libepoxy-dev` — required for virglrenderer build
- `meson` / `ninja-build` — virglrenderer build system
- `curl` — to fetch `unpack_bootimg` from AOSP

---

## 3. Build virglrenderer from Source

L4T ships virglrenderer 0.9.1 but crosvm requires >=1.0:

```bash
cd ~/work
git clone https://gitlab.freedesktop.org/virgl/virglrenderer.git
cd virglrenderer
git checkout main  # or latest stable tag

meson setup builddir -Dprefix=/usr/local
ninja -C builddir
sudo ninja -C builddir install
sudo ldconfig

# Verify
pkg-config --modversion virglrenderer
# Should show >= 1.0.0
```

---

## 4. Build crosvm

Same as RPi5 doc but note:
- No need to install `libminijail-dev` — crosvm builds minijail from its submodule
- Ensure `PKG_CONFIG_PATH` includes `/usr/local/lib/aarch64-linux-gnu/pkgconfig` for virglrenderer

```bash
cd ~/work
git clone https://chromium.googlesource.com/crosvm/crosvm
cd crosvm
git submodule update --init

# Build with virgl_renderer feature
cargo build --release --features "virgl_renderer"
```

---

## 5. Patch vbmeta (Disable AVB Hashtree Verification)

Android 17 dropped support for `androidboot.veritymode=disabled` kernel parameter.
You must patch the vbmeta image header directly:

```bash
cd ~/work/aosp/images

# Set flags byte at offset 120 to 0x03 (disable verification + hashtree)
printf '\x00\x00\x00\x03' | dd of=vbmeta.img seek=120 bs=1 count=4 conv=notrunc
printf '\x00\x00\x00\x03' | dd of=vbmeta_system.img seek=120 bs=1 count=4 conv=notrunc
```

Flag bits:
- Bit 0 (0x01): Disable dm-verity verification
- Bit 1 (0x02): Disable hashtree descriptor processing
- Combined (0x03): Fully disable AVB enforcement

> **Note**: The `androidboot.veritymode=disabled` param is still passed for other code paths
> but the actual enforcement is controlled by these vbmeta header flags.

---

## 6. Patch Vendor Image (Disable Encryption in super.img)

The `/data` mount encryption flags live in `/vendor/etc/fstab.cf.*` **inside the vendor
partition within super.img** (not in the ramdisk). On Orin, the vendor filesystem is `erofs`
which can't be mounted directly. Use binary patching:

### 6.1. Find vendor_a offset in super partition

```bash
cd ~/work/aosp
LOOP=$(sudo losetup --show -fP composite.img)

# Parse LP (Logical Partition) metadata to find vendor_a offset
sudo python3 -c "
import struct
with open('/dev/${LOOP##*/}p10','rb') as f:
  f.seek(12288); hdr=f.read(256)
  mag=struct.unpack_from('<I',hdr,0)[0]
  print(f'LP metadata magic: {mag:#x}')
  hs=struct.unpack_from('<I',hdr,8)[0]
  po,pn,ps=struct.unpack_from('<III',hdr,80)
  eo,en,es=struct.unpack_from('<III',hdr,92)
  tb=12288+hs
  exts=[]
  for i in range(en):
    f.seek(tb+eo+i*es); d=f.read(es)
    num_sectors=struct.unpack_from('<Q',d,0)[0]
    target_type=struct.unpack_from('<I',d,8)[0]
    target_data=struct.unpack_from('<Q',d,12)[0]
    exts.append((num_sectors, target_type, target_data))
  for i in range(pn):
    f.seek(tb+po+i*ps); d=f.read(ps)
    nm=d[:36].split(b'\x00')[0].decode()
    attr,fe,ne,gi=struct.unpack_from('<IIII',d,36)
    if ne>0 and exts[fe][1]==0:
      start_sector=exts[fe][2]
      size_sectors=exts[fe][0]
      print(f'{nm}: offset={start_sector*512} size={size_sectors*512} ({size_sectors*512//1048576}MB)')
"
```

Example output:
```
LP metadata magic: 0x414c5030
vendor_a: offset=1468006400 size=291229696 (277MB)
```

### 6.2. Extract vendor_a

```bash
# Use the offset value from step 6.1 (e.g., 1468006400)
VENDOR_OFFSET=1468006400
VENDOR_SIZE=291229696

sudo dd if=${LOOP}p10 of=/tmp/vendor_a.img \
    bs=1M skip=$((VENDOR_OFFSET/1048576)) count=$((VENDOR_SIZE/1048576))
```

### 6.3. Verify encryption flags exist

```bash
grep -c 'metadata_encryption' /tmp/vendor_a.img
grep -c 'fileencryption' /tmp/vendor_a.img
# Both should return >0
```

### 6.4. Binary-patch encryption flags

Since erofs can't be mounted/edited, we rename the first character of each flag so
`fs_mgr` won't recognize them. Same byte length = no filesystem corruption:

```bash
sudo perl -pi -e '
    s/metadata_encryption/xetadata_encryption/g;
    s/fileencryption/xileencryption/g;
    s/wrappedkey/xrappedkey/g;
    s/keydirectory/xeydirectory/g
' /tmp/vendor_a.img

# Verify all removed
grep -c 'metadata_encryption\|fileencryption\|wrappedkey\|keydirectory' /tmp/vendor_a.img
# Should return 0
```

### 6.5. Write patched vendor_a back into super

```bash
sudo dd if=/tmp/vendor_a.img of=${LOOP}p10 \
    bs=1M seek=$((VENDOR_OFFSET/1048576)) conv=notrunc
```

### 6.6. Clean up

```bash
sudo losetup -d $LOOP
rm /tmp/vendor_a.img
```

---

## 7. Prepare Metadata and Userdata

After patching vendor, re-format metadata and userdata to clear any encryption state:

```bash
LOOP=$(sudo losetup --show -fP composite.img)

# Format metadata as ext4
sudo mkfs.ext4 -F -L metadata ${LOOP}p9

# Format userdata as f2fs (matching fstab expectations)
sudo mkfs.f2fs -f -O project_quota,extra_attr -l userdata ${LOOP}p11

sudo losetup -d $LOOP
```

> **Note**: Do this every time you re-patch vendor or if you hit
> `enablefilecrypto_failed` during boot.

---

## 8. Boot Command (Orin Nano)

### Working boot command (with vsock ADB):

```bash
cd ~/work/aosp

# Pre-requisite: vsock modules loaded (see Section 9)
sudo ../crosvm/target/release/crosvm run \
    -m 1024 --cpus 2 --disable-sandbox \
    --cid 3 \
    --serial type=stdout,stdin=true \
    -p "console=ttyAMA0 loglevel=0 \
        androidboot.hardware=cutf_cvm \
        androidboot.boot_devices=10000.pci \
        androidboot.slot_suffix=_a \
        androidboot.fstab_suffix=cf.f2fs.hctr2 \
        androidboot.verifiedbootstate=orange \
        androidboot.force_normal_boot=1 \
        androidboot.veritymode=disabled \
        androidboot.vbmeta.device_state=unlocked \
        androidboot.vbmeta.avb_version=1.1 \
        androidboot.vbmeta.hash_alg=sha256 \
        androidboot.vbmeta.invalidate_on_error=yes \
        panic=0 \
        androidboot.enable_console=1 \
        androidboot.vendor.apex.com.android.hardware.gatekeeper=com.android.hardware.gatekeeper.nonsecure.apex \
        androidboot.vendor.apex.com.android.hardware.keymint=com.android.hardware.keymint.rust_nonsecure.apex \
        androidboot.vendor.apex.com.android.hardware.graphics.composer=com.android.hardware.graphics.composer.ranchu.apex \
        androidboot.vendor.apex.com.google.emulated.camera.provider.hal=com.google.emulated.camera.provider.hal.apex \
        androidboot.lcd_density=320 \
        androidboot.hardware.egl=angle \
        androidboot.hardware.gralloc=minigbm \
        androidboot.hardware.hwcomposer=ranchu \
        androidboot.hardware.vulkan=pastel \
        androidboot.opengles.version=196609 \
        androidboot.hardware.hwcomposer.mode=noop \
        androidboot.hardware.hwcomposer.display_finder_mode=noop \
        androidboot.cpuvulkan.version=4202496 \
        androidboot.hw_timeout_multiplier=50 \
        androidboot.adb.enabled=1 \
        service.adb.tcp.port=5555 \
        androidboot.console=ttyAMA0" \
    --block path=composite.img \
    --initrd merged_ramdisk.img \
    boot_unpacked/kernel
```

### Key boot parameter differences from RPi5:

| Parameter | RPi5 | Orin Nano |
|-----------|-------|-----------|
| `-m` | 1024 | 1024 (can use 2048 with 8GB host) |
| `--cid 3` | Yes (built-in) | Yes (after building vsock modules from L4T source) |
| `androidboot.enable_console=1` | Not needed | Required for serial shell |
| `loglevel` | 0 | 0 (use 7 for debugging) |

---

## 9. ADB Access (via vsock)

ADB works over vsock after building the `vhost_vsock` module from NVIDIA's L4T kernel source.

### 9.1. Download NVIDIA L4T kernel source

```bash
cd ~/work

# Download L4T R36.4.4 public sources
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.4/sources/public_sources.tbz2

# Extract
tar xjf public_sources.tbz2
cd Linux_for_Tegra/source
tar xjf kernel_src.tbz2
cd kernel/kernel-jammy-src
```

### 9.2. Configure and build vsock modules

```bash
# Copy running kernel config
cp /proc/config.gz .
gunzip config.gz
mv config .config

# Enable vsock modules
scripts/config --module CONFIG_VSOCKETS
scripts/config --module CONFIG_VHOST
scripts/config --module CONFIG_VHOST_NET
scripts/config --module CONFIG_VHOST_VSOCK
scripts/config --module CONFIG_VIRTIO_VSOCKETS
scripts/config --module CONFIG_VIRTIO_VSOCKETS_COMMON

# CRITICAL: Set localversion to match running kernel vermagic
scripts/config --set-str CONFIG_LOCALVERSION "-tegra"

# Build
make olddefconfig
make modules_prepare
make -j$(nproc) M=net/vmw_vsock modules
make -j$(nproc) M=drivers/vhost modules

# Verify vermagic matches (must be "5.15.148-tegra")
modinfo net/vmw_vsock/vsock.ko | grep vermagic
```

> **CRITICAL**: If vermagic doesn't include `-tegra`, modules will cause 100% CPU spin
> on the vhost kernel thread. The `CONFIG_LOCALVERSION="-tegra"` setting is mandatory.

### 9.3. Install and load modules

```bash
sudo cp net/vmw_vsock/vsock.ko /lib/modules/$(uname -r)/extra/
sudo cp net/vmw_vsock/vmw_vsock_virtio_transport_common.ko /lib/modules/$(uname -r)/extra/
sudo cp net/vmw_vsock/vsock_loopback.ko /lib/modules/$(uname -r)/extra/
sudo cp drivers/vhost/vhost.ko /lib/modules/$(uname -r)/extra/
sudo cp drivers/vhost/vhost_vsock.ko /lib/modules/$(uname -r)/extra/
sudo cp drivers/vhost/vhost_net.ko /lib/modules/$(uname -r)/extra/

sudo depmod -a

# Load in order
sudo modprobe vsock
sudo modprobe vhost
sudo modprobe vmw_vsock_virtio_transport_common
sudo modprobe vhost_vsock

# Verify loaded, no CPU spike
lsmod | grep vsock
top -bn1 | head -5
```

### 9.4. Connect ADB

```bash
# After VM boots (~60-90 seconds):
adb connect vsock:3
adb shell
```

That's it — same as RPi5. No TAP networking or DHCP required.

### 9.5. Make module loading persistent across reboot

```bash
echo -e "vsock\nvhost\nvmw_vsock_virtio_transport_common\nvhost_vsock" | \
    sudo tee /etc/modules-load.d/vsock.conf
```

---

## 10. Display Output on DP

### 10.0. GPU/Display Architecture

```
Android App → SurfaceFlinger → HWComposer (ranchu/client)
    → Gralloc (minigbm) → virtio-gpu (guest) → crosvm
    → Wayland → Weston → Tegra DRM → DP output
```

The Orin Nano's Tegra GPU is a platform device (not PCIe), so VFIO passthrough is not
possible. Display output goes through crosvm's virtio-gpu device to a Wayland compositor.

### 10.1. EGL Backend Compatibility (IMPORTANT)

The prebuilt Cuttlefish images ship three EGL options:

| `androidboot.hardware.egl=` | Library | Protocol | Status on Orin |
|------------------------------|---------|----------|----------------|
| `emulation` | `/vendor/lib64/egl/libEGL_emulation.so` | **gfxstream** | ❌ Crashes — requires `--gpu backend=gfxstream` + gfxstream_backend host lib (not available on L4T) |
| `angle` | ANGLE (bundled in system) | SwiftShader (CPU) | ✅ Works — software rendering, slow but functional |
| `mesa` | Not present in prebuilt images | virgl | ❌ Missing — would need AOSP source build with mesa |

**Key finding**: virglrenderer on the host works (GLES 3.2, GLSL 430 on Tegra), but there's
no matching guest-side EGL that speaks the virgl protocol. The `emulation` EGL speaks gfxstream
(different protocol), causing `EGL_NOT_INITIALIZED → SIGABRT` in SurfaceFlinger.

**Working configuration**: `egl=angle` with `--gpu backend=2d` — ANGLE renders in software
(CPU), HWComposer scans out the framebuffer via virtio-gpu 2D to the Wayland display.

**Production fix**: Build gfxstream_backend from AOSP source, install on host, rebuild crosvm
with `--features "gpu,gfxstream"`, then use `egl=emulation` + `--gpu backend=gfxstream` for
hardware-accelerated GPU rendering via Tegra.

### 10.2. Headless (no display, serial/ADB only)

Omit `--gpu` and `--wayland-sock` from the crosvm command. Most stable configuration.
Use `androidboot.hardware.hwcomposer.mode=noop` in this mode.

### 10.3. Display via Weston (WORKING — verified)

L4T ships its own Weston (`nvidia-l4t-weston`). Do NOT install Ubuntu's weston package.

**Step 1: Start Weston (from SSH session, not the DP console)**

```bash
export XDG_RUNTIME_DIR=/tmp/weston-run
mkdir -p $XDG_RUNTIME_DIR && chmod 700 $XDG_RUNTIME_DIR

# Launch Weston on DRM (takes over DP output — shows grey pattern)
weston --backend=drm-backend.so --current-mode 2>&1 &
sleep 2

# Verify socket created
ls $XDG_RUNTIME_DIR/wayland-*
# Should show: wayland-0  wayland-0.lock
```

**Step 2: Boot crosvm with display**

```bash
cd ~/work/aosp

sudo ../crosvm/target/release/crosvm run \
    -m 2048 --cpus 4 --disable-sandbox \
    --cid 3 \
    --wayland-sock /tmp/weston-run/wayland-0 \
    --gpu backend=2d,displays=[[mode=windowed[1280,720]]] \
    --serial type=stdout,stdin=true \
    -p "console=ttyAMA0 loglevel=0 \
        androidboot.hardware=cutf_cvm \
        androidboot.boot_devices=10000.pci \
        androidboot.slot_suffix=_a \
        androidboot.fstab_suffix=cf.f2fs.hctr2 \
        androidboot.verifiedbootstate=orange \
        androidboot.force_normal_boot=1 \
        androidboot.veritymode=disabled \
        androidboot.vbmeta.device_state=unlocked \
        androidboot.vbmeta.avb_version=1.1 \
        androidboot.vbmeta.hash_alg=sha256 \
        androidboot.vbmeta.invalidate_on_error=yes \
        panic=0 \
        androidboot.enable_console=1 \
        androidboot.vendor.apex.com.android.hardware.gatekeeper=com.android.hardware.gatekeeper.nonsecure.apex \
        androidboot.vendor.apex.com.android.hardware.keymint=com.android.hardware.keymint.rust_nonsecure.apex \
        androidboot.vendor.apex.com.android.hardware.graphics.composer=com.android.hardware.graphics.composer.ranchu.apex \
        androidboot.vendor.apex.com.google.emulated.camera.provider.hal=com.google.emulated.camera.provider.hal.apex \
        androidboot.lcd_density=320 \
        androidboot.hardware.egl=angle \
        androidboot.hardware.gralloc=minigbm \
        androidboot.hardware.hwcomposer=ranchu \
        androidboot.hardware.vulkan=pastel \
        androidboot.opengles.version=196609 \
        androidboot.hardware.hwcomposer.mode=client \
        androidboot.hardware.hwcomposer.display_finder_mode=drm \
        androidboot.cpuvulkan.version=4202496 \
        androidboot.hw_timeout_multiplier=50 \
        androidboot.adb.enabled=1 \
        service.adb.tcp.port=5555 \
        androidboot.console=ttyAMA0" \
    --block path=composite.img \
    --initrd merged_ramdisk.img \
    boot_unpacked/kernel
```

**Key display parameters (different from headless boot):**

| Parameter | Headless | With Display |
|-----------|----------|-------------|
| `--wayland-sock` | omitted | `/tmp/weston-run/wayland-0` |
| `--gpu` | omitted | `backend=2d,displays=[[mode=windowed[1280,720]]]` |
| `hwcomposer.mode` | `noop` | `client` |
| `display_finder_mode` | `noop` | `drm` |

> **Note**: `--wayland-sock` is REQUIRED. crosvm does NOT read `WAYLAND_DISPLAY` or
> `XDG_RUNTIME_DIR` env vars — the socket path must be passed explicitly on the command line.

### 10.4. GPU Backend Options Tested

| Backend | Command | Result |
|---------|---------|--------|
| `backend=2d` | `--gpu backend=2d,displays=[[mode=windowed[1280,720]]]` | ✅ Works — display output on DP, software rendering (slow) |
| `backend=virglrenderer` | `--gpu backend=virglrenderer,egl=true,gles=true,...` | ⚠️ Display appears (black/boot anim) but floods `TransferToHost3d` errors |
| `backend=gfxstream` | `--gpu backend=gfxstream,...` | ❌ Build fails — requires `gfxstream_backend` host library (not on L4T) |

### 10.5. Production GPU acceleration (gfxstream — TODO)

For native GPU performance, build gfxstream_backend from AOSP source:

```bash
cd ~/work
git clone https://android.googlesource.com/platform/hardware/google/gfxstream
cd gfxstream
# Build with cmake (needs EGL, GLES, Vulkan dev libs on host)
# Install gfxstream_backend.pc to /usr/local/lib/pkgconfig/

# Rebuild crosvm with gfxstream
cd ~/work/crosvm
PKG_CONFIG_PATH=/usr/local/lib/pkgconfig cargo build --release --features "gpu,gfxstream"

# Boot with gfxstream
# --gpu backend=gfxstream,egl=true,gles=true,displays=[[mode=windowed[1280,720]]]
# androidboot.hardware.egl=emulation  (uses existing Cuttlefish gfxstream EGL)
```

This enables hardware-accelerated rendering: Android → gfxstream protocol → Tegra GPU → display.

### 10.6. DRM/Render Nodes on Orin Nano

```
/dev/dri/card0      → host1x (Tegra display controller)
/dev/dri/card1      → nv_platform (Tegra display)
/dev/dri/renderD128 → host1x
/dev/dri/renderD129 → nv_platform (tegra234-display)
```

The Tegra GPU is `nvgpu` (integrated, not discrete NVIDIA dGPU). Modules: `nvidia_drm`,
`nvidia_modeset`, `tegra_drm`, `nvgpu`.

---

## 11. Troubleshooting (Orin-Specific)

| # | Issue | Cause | Fix |
|---|-------|-------|-----|
| 1 | `vhost_vsock` not found | L4T kernel doesn't include the module | Build from L4T kernel source with `CONFIG_LOCALVERSION="-tegra"` (Section 9) |
| 2 | `Operation not permitted` on `--net` | crosvm can't open /dev/net/tun | Run crosvm with `sudo` |
| 3 | `Unknown androidboot.veritymode: disabled` → mount failure | Android 17 dropped kernel param support | Patch vbmeta.img header flags (Section 5) |
| 4 | `/data` read-only → `enablefilecrypto_failed` | Encryption flags in vendor fstab inside super.img | Binary-patch vendor_a.img (Section 6) |
| 5 | `sed -i` fails on vendor image | sed uses rename which fails on some filesystems | Use `perl -pi -e` instead |
| 6 | No serial shell after boot | `console` service not started | Add `androidboot.enable_console=1` to cmdline |
| 7 | No DHCP lease from VM | `virt_wifi` wraps virtio-net as WiFi | Add `module_blacklist=virt_wifi` to cmdline |
| 8 | `ioctl(TUNSETIFF): Device or resource busy` | tap0 already exists | Ignore — it's already created |
| 9 | BootAnimation GPU errors (TransferToHost3d) | virglrenderer can't handle ANGLE's 3D buffers | Use `--gpu backend=2d` instead of `virglrenderer` |
| 10 | HAL services crashing (sensors, light, uwb, etc.) | Cuttlefish HALs need host-side daemons not running | Crashes block boot completion. Stop with `adb shell setprop ctl.stop <service>` or disable in source |
| 11 | KVM permission denied | User not in kvm group | `sudo usermod -aG kvm nano && newgrp kvm` |
| 12 | virglrenderer too old (0.9.1) | L4T system package outdated | Build from source (Section 3) |
| 13 | `erofs` filesystem can't be mounted | L4T kernel lacks erofs support | Use binary patching instead of mount+sed (Section 6.4) |
| 14 | `failed to open display: unsupported` | crosvm can't find Wayland socket | Pass `--wayland-sock /tmp/weston-run/wayland-0` explicitly — env vars don't work |
| 15 | SurfaceFlinger SIGABRT (`EGL_NOT_INITIALIZED`, GFXSTREAM errors) | `egl=emulation` requires gfxstream backend, not virglrenderer | Use `egl=angle` (software) or build gfxstream_backend for hardware accel |
| 16 | `egl=mesa` → SurfaceFlinger not starting | Mesa virgl EGL not included in prebuilt Cuttlefish images | Use `egl=angle` or build AOSP from source with mesa |
| 17 | `sys.boot_completed` never returns 1 (standalone crosvm) | HAL services crash-looping (sensors, light, uwb, etc.) because no host daemons are running — this is the fundamental limitation of standalone crosvm, which is why run_cvd is required | Use run_cvd (Section 12) which provides host daemons. Standalone crosvm will **never** achieve boot_completed |
| 18 | `nvidia-l4t-weston` conflicts with Ubuntu weston | L4T ships own Weston package | Use the L4T Weston (`which weston`), don't `apt install weston` |
| 19 | `no valid GPU path provided` (virglrenderer) | No `rendernode=` specified | Non-fatal — virglrenderer finds EGL automatically via Tegra. GLES 3.2 + GLSL 430 confirmed |
| 20 | `apexd-bootstrap` SIGABRT → reboot loop (`bootstrap-apexd-failed`) | Two APEX packages for same module (e.g. gatekeeper: `cf_remote.apex` + `nonsecure.apex`) — no selection property in cmdline | Add `androidboot.vendor.apex.*` properties to KCMDLINE (Section 12.3) |
| 21 | VM reboot loop (`sys.init.updatable_crashing=1`) after run_cvd boot | `vendor.light-cuttlefish` HAL crashes because `vsock_lights_port=6900` property tells it to connect to host daemon that doesn't exist. After 4+ crashes, rescue party reboots the VM. Accumulates across reboots making it progressively worse | Add `androidboot.disable_rescue_party=true` to KCMDLINE. Light HAL still crashes harmlessly but won't trigger VM reboots. Boot completes at ~26s, stays up |
| 22 | Touch not working (evdev not passed to VM) | Touch device permissions `crw-rw----` (root:input) at crosvm launch time — wrapper's `[ -r ]` check fails silently, skips `--input` | Create udev rule: `SUBSYSTEM=="input", ATTRS{idVendor}=="0eef", MODE="0666"` in `/etc/udev/rules.d/99-egalax-touch.rules`. Must be set BEFORE run_cvd launch |
| 23 | Touch device number changes across reboots (event5 → event1) | USB enumeration order varies | Use auto-detect in wrapper: `grep -l eGalax /sys/class/input/event*/device/name` instead of hardcoded path |
| 24 | `--novhost_user_vsock` flag has no effect | assemble_cvd ignores the flag — `vhost_user_vsock` stays `true` in config | Patch config manually after assemble: set `vhost_user_vsock=false` (Section 12.2.1). Without this, `vhost_device_vsock` crash-loops and blocks VM launch |
| 25 | `enable_gpu_vhost_user` stays `true` despite `--gpu_mode=gfxstream` | assemble_cvd doesn't set this to `false` for inline mode | Patch config manually after assemble: set `enable_gpu_vhost_user=false` (Section 12.2.1). Without this, crosvm tries to connect to a non-existent vhost-user-gpu socket |
| 26 | `persistent_composite.img` has no partitions after fresh assemble | assemble creates it as composite_disk protobuf format (not raw GPT) — crosvm can't parse it on aarch64 | Must recreate as raw GPT after EVERY `assemble_cvd` run (Section 12.5). Without this, `/dev/block/by-name/frp` is missing → `PersistentDataBlockService` timeout → `system_server` crash → `updatable_crashing=1` → boot never completes |
| 27 | `pflash.img truncate --size=-36M` fails during assemble | `bootloader.crosvm` is 42MB kernel instead of 719KB u-boot — pflash size calculation goes negative | Restore original u-boot (719KB) as `bootloader.crosvm` BEFORE assemble, replace with kernel AFTER (Section 12.2) |
| 28 | Weston GL renderer fails: `EGL_NOT_INITIALIZED`, `DRI2: gbm device using incorrect/incompatible backend` | After reboot, `nvidia-drm` loads before `tegra`, making card0=nvidia-drm (GBM backend="nvidia"). NVIDIA EGL rejects the "nvidia" GBM backend — only accepts "tegra". Render node numbering inverts (renderD128=nvidia-drm instead of tegra) | Swap render device nodes before starting Weston: `sudo mv /dev/dri/renderD128 /dev/dri/renderD128_bak && sudo mv /dev/dri/renderD129 /dev/dri/renderD128 && sudo mv /dev/dri/renderD128_bak /dev/dri/renderD129`. See Section 13.2.1 for auto-detect script. Ref: [NVIDIA forum](https://forums.developer.nvidia.com/t/348933) |
| 29 | gfxstream `Failed to find exactly 1 GLES 2.x config: found 0` → renderer fails to initialize | NVIDIA's bare EGL display (without X11/Wayland) only has `EGL_PBUFFER_BIT` configs, no `EGL_WINDOW_BIT`. gfxstream's config query uses default `EGL_SURFACE_TYPE` which requires `EGL_WINDOW_BIT` | Deploy `egl_config_shim.so` (LD_PRELOAD) that adds `EGL_SURFACE_TYPE=EGL_PBUFFER_BIT` to queries missing it. Added to `bin/crosvm` wrapper. See Section 13.2.2 |

---

## Quick Reference: Complete Orin Nano Setup from Scratch

```bash
# Assumes: fresh L4T Ubuntu 22.04 on Orin Nano with KVM enabled

# 1. Install deps (Section 2 + RPi5 doc Section 2)
# 2. Build virglrenderer from source (Section 3)
# 3. Build crosvm (Section 4)
# 4. Download + extract Cuttlefish images (RPi5 doc Section 4-5)
# 5. Create composite disk (RPi5 doc Section 6)
# 6. Patch vbmeta headers (Section 5)
# 7. Merge ramdisks + patch init symlink (RPi5 doc Section 5)
# 8. Patch vendor in super.img (Section 6)
# 9. Format metadata + userdata (Section 7)
# 10. Build & load vsock modules (Section 9.1-9.3)
# 11. Start Weston for display (Section 10.3 Step 1)
# 12. Boot with sudo + --cid 3 + --wayland-sock + --gpu backend=2d (Section 10.3 Step 2)
# 13. Connect ADB: adb connect vsock:3 (Section 9.4)
# 14. Stop crashing HALs if boot doesn't complete (Section 11, #17)
```

---

## 12. Boot via run_cvd (Full Cuttlefish Infrastructure)

Instead of launching crosvm manually, use `run_cvd` / `assemble_cvd` to get all Cuttlefish host daemons (modem_simulator, sensors_simulator, adb proxy, logcat, etc.).

### 12.1 Build CF Host Tools via Bazel

```bash
cd ~/work/android-cuttlefish
# Patch CrosvmManager::IsSupported() to return true (Orin's KVM works but isn't detected)
# File: base/cvd/cuttlefish/host/libs/vm_manager/crosvm_manager.cpp
tools/buildutils/build_packages.sh
# Copy all binaries from bazel-bin to ~/work/aosp_imgs/bin/
```

### 12.2 Assemble

**CRITICAL**: `bootloader.crosvm` must be the original u-boot (719KB) during assemble —
the pflash size calculation uses bootloader size and breaks with the 42MB kernel.
Restore kernel as bootloader AFTER assemble completes (see step 2 in Section 12.3).

```bash
# Ensure u-boot is in place (NOT the kernel)
cp etc/bootloader_aarch64/bootloader.crosvm.uboot etc/bootloader_aarch64/bootloader.crosvm

# Remove prior state
rm -rf cuttlefish .cuttlefish_config.json

cd ~/work/aosp_imgs
ANDROID_HOST_OUT=$PWD HOME=$PWD ./bin/assemble_cvd \
  --config=phone --gpu_mode=gfxstream --cpus=4 --memory_mb=4096 \
  --vm_manager=crosvm --noenable_host_network --novhost_user_vsock \
  --noenable_wifi < /dev/null
```

> **WARNING — Assemble flags that DON'T work as expected:**
>
> | Flag | Expected | Actual | Fix |
> |------|----------|--------|-----|
> | `--novhost_user_vsock` | Sets `vhost_user_vsock=false` | **Leaves it `true`** | Patch config manually (Section 12.2.1) |
> | `--gpu_mode=gfxstream` | Sets `enable_gpu_vhost_user=false` for inline mode | **Leaves it `true`** (launches separate vhost-user-gpu process) | Patch config manually (Section 12.2.1) |
>
> Without these patches: `vhost_device_vsock` crash-loops blocking VM launch,
> and the GPU device process fails to connect to a non-existent vhost-user socket.

#### 12.2.1 Post-Assemble Config Patches (Required)

After assemble completes, patch both config files:

```bash
# Patch vhost_user_vsock and enable_gpu_vhost_user in both configs
python3 -c "
import json, glob
for f in glob.glob('cuttlefish/**/cuttlefish_config.json', recursive=True):
    with open(f) as fh: cfg = json.load(fh)
    for inst in cfg.get('instances', [cfg]):
        inst['vhost_user_vsock'] = False
        inst['enable_gpu_vhost_user'] = False
    with open(f, 'w') as fh: json.dump(cfg, fh, indent=4)
    print(f'Patched: {f}')
"
```

With `enable_gpu_vhost_user=false`, gfxstream rendering happens **inline** in the main
crosvm process. No separate GPU device process is launched, and **Section 13.4 (prebuilts/crosvm
wrapper) is NOT needed**. The main crosvm wrapper (Section 12.3) handles wayland redirection.

### 12.3 crosvm Wrapper Script (Critical)

CrosvmManager passes `--bios bootloader.crosvm` (U-Boot), which can't parse the composite disk GPT. Replace `bin/crosvm` with a wrapper that converts `--bios` to direct kernel boot with `--initrd`:

1. Copy the real binary: `mv bin/crosvm bin/crosvm.real`
2. Replace `bootloader.crosvm` with the actual kernel Image:
   ```bash
   cp etc/bootloader_aarch64/bootloader.crosvm etc/bootloader_aarch64/bootloader.crosvm.orig
   cp kernel_Image etc/bootloader_aarch64/bootloader.crosvm
   ```
3. Create combined ramdisk with bootconfig:
   ```bash
   # Extract bootconfig text from assemble_cvd output
   cat cuttlefish/instances/cvd-1/internal/bootconfig  # 1978 bytes, 40 lines
   # Append bootconfig trailer to ramdisk
   cat combined_ramdisk.img bootconfig_text > /tmp/initrd_with_bootconfig.img
   # (use proper bootconfig append tool or manual trailer with magic)
   ```
4. Create `bin/crosvm` wrapper:

```bash
#!/bin/bash
ulimit -n 65536 2>/dev/null
REAL="$(dirname "$(readlink -f "$0")")/crosvm.real"
SCRIPT_DIR="$(dirname "$(readlink -f "$0")")"
BASE_DIR="$(dirname "$SCRIPT_DIR")"
INITRD="$BASE_DIR/initrd_with_bootconfig.img"
KCMDLINE="printk.devkmsg=on audit=1 panic=-1 8250.nr_uarts=1 binder.impl=rust cma=0 firmware_class.path=/vendor/etc/ loop.max_part=7 init=/init androidboot.boot_devices=10000.pci androidboot.fstab_suffix=cf.f2fs.hctr2 androidboot.force_normal_boot=1 androidboot.serialno=CUTTLEFISHCVD01 androidboot.hardware=cutf_cvm androidboot.slot_suffix=_a androidboot.verifiedbootstate=orange androidboot.veritymode=disabled androidboot.vbmeta.device_state=unlocked androidboot.vbmeta.avb_version=1.1 androidboot.vbmeta.hash_alg=sha256 androidboot.vbmeta.invalidate_on_error=yes androidboot.hardware.egl=emulation androidboot.hardware.gralloc=minigbm androidboot.hardware.hwcomposer=ranchu androidboot.hardware.vulkan=ranchu androidboot.opengles.version=196609 androidboot.hardware.gltransport=virtio-gpu-asg androidboot.lcd_density=320 androidboot.setupwizard_mode=DISABLED androidboot.hypervisor.vm.supported=0 androidboot.hypervisor.protected_vm.supported=0 androidboot.enable_bootanimation=1 androidboot.hardware.hwcomposer.display_finder_mode=drm androidboot.hardware.hwcomposer.display_framebuffer_format=rgba androidboot.serialconsole=0 androidboot.ddr_size=4915MB androidboot.modem_simulator_ports=9600 androidboot.vendor.apex.com.android.hardware.gatekeeper=com.android.hardware.gatekeeper.nonsecure.apex androidboot.vendor.apex.com.android.hardware.keymint=com.android.hardware.keymint.rust_nonsecure.apex androidboot.vendor.apex.com.android.hardware.graphics.composer=com.android.hardware.graphics.composer.ranchu.apex androidboot.vendor.apex.com.google.emulated.camera.provider.hal=com.google.emulated.camera.provider.hal.apex androidboot.vsock_lights_port=6900 androidboot.vsock_lights_cid=3 androidboot.disable_rescue_party=true androidboot.bluetooth.disabled=true"

export XDG_RUNTIME_DIR=/tmp/weston-run
export LD_PRELOAD=${BASE_DIR}/egl_config_shim.so

ARGS=("$@")
NEW_ARGS=()
KERNEL_PATH=""
i=0

while [ $i -lt ${#ARGS[@]} ]; do
    arg="${ARGS[$i]}"
    if [[ "$arg" == "--bios="* ]]; then
        KERNEL_PATH="${arg#--bios=}"
    elif [[ "$arg" == "--bios" ]]; then
        i=$((i+1))
        KERNEL_PATH="${ARGS[$i]}"
    else
        NEW_ARGS+=("$arg")
    fi
    i=$((i+1))
done

# Strip ALL --vhost-user=* args (mac80211, input, etc.) — wrapper handles GPU inline
# Strip --sound=* and --vhost-user-connect-timeout-ms=* (not supported without vhost-user)
FILTERED_ARGS=()
for a in "${NEW_ARGS[@]}"; do
    [[ "$a" == --vhost-user=* ]] && continue
    [[ "$a" == --vhost-user-mac80211-hwsim=* ]] && continue
    [[ "$a" == --sound=* ]] && continue
    [[ "$a" == --vhost-user-connect-timeout-ms=* ]] && continue
    FILTERED_ARGS+=("$a")
done
NEW_ARGS=("${FILTERED_ARGS[@]}")

if [ -n "$KERNEL_PATH" ]; then
    echo "[crosvm-wrapper] Converting --bios to direct kernel boot" >&2
    rm -f /tmp/cf_avd_1000/cvd-1/internal/crosvm_control.sock 2>/dev/null

    # Redirect wayland to Weston's real socket
    FINAL_ARGS=()
    for a in "${NEW_ARGS[@]}"; do
        if [[ "$a" == "--wayland-sock="* ]]; then
            FINAL_ARGS+=("--wayland-sock=/tmp/weston-run/wayland-0")
        elif [[ "$a" == *'"windowed":[720,1280]'* ]]; then
            FINAL_ARGS+=("${a//\"windowed\":\[720,1280\]/\"windowed\":[1280,720]}")
        else
            FINAL_ARGS+=("$a")
        fi
    done

    # Auto-detect eGalax touch device
    TOUCH_ARGS=()
    TOUCH_DEV=$(grep -l eGalax /sys/class/input/event*/device/name 2>/dev/null | head -1 | sed 's|/sys/class/input/|/dev/input/|;s|/device/name||')
    if [ -n "$TOUCH_DEV" ] && [ -r "$TOUCH_DEV" ]; then
        echo "[crosvm-wrapper] Adding eGalax touch input $TOUCH_DEV" >&2
        TOUCH_ARGS+=(--input "evdev[path=$TOUCH_DEV]")
    fi

    export XDG_RUNTIME_DIR=/tmp/weston-run
    exec "$REAL" "${FINAL_ARGS[@]}" "${TOUCH_ARGS[@]}" --initrd "$INITRD" -p "$KCMDLINE" "$KERNEL_PATH"
fi

exec "$REAL" "$@"
```

> **Key KCMDLINE properties (beyond standard boot params):**
> - `androidboot.vendor.apex.*` — Resolves dual-APEX conflicts that cause `apexd-bootstrap` SIGABRT
> - `androidboot.vsock_lights_port=6900` / `vsock_lights_cid=3` — Light HAL connectivity (from bootconfig)
> - `androidboot.disable_rescue_party=true` — Prevents reboot loop when light HAL crashes (no host daemon)
> - `androidboot.bluetooth.disabled=true` — Disables BT framework initialization (added by wrapper)
>
> **INITRD path**: Uses `$BASE_DIR/initrd_with_bootconfig.img` (persists across reboots, not `/tmp`)
>
> **Wrapper stripping**: The wrapper strips ALL `--vhost-user=*` args (mac80211, input devices),
> `--sound=*`, and `--vhost-user-connect-timeout-ms=*`. These are passed by run_cvd expecting
> vhost-user devices that don't exist when `enable_gpu_vhost_user=false` and `vhost_user_vsock=false`.
> Without stripping, crosvm fails with socket connection errors.
>
> **Wayland redirection**: With `enable_gpu_vhost_user=false`, the GPU is inline in the main crosvm
> process. The wrapper redirects `--wayland-sock` from the internal CF socket to `/tmp/weston-run/wayland-0`
> and swaps display dimensions from portrait (720×1280) to landscape (1280×720). **Section 13.4
> (prebuilts/crosvm wrapper) is NOT needed** in this configuration.
>
> **EGL config shim**: The wrapper sets `LD_PRELOAD=${BASE_DIR}/egl_config_shim.so` to fix
> gfxstream's GLES config detection on NVIDIA's bare EGL (Section 13.2.2). Without this,
> gfxstream fails with "Failed to find exactly 1 GLES 2.x config: found 0".

### 12.4 Create Raw Disk Image (Replace Composite Disk)

CrosvmManager generates `os_composite.img` (composite_disk protobuf format). crosvm may not parse it correctly on aarch64. Create a raw GPT disk instead:

```bash
INST=~/work/aosp_imgs/cuttlefish/instances/cvd-1
# Use sgdisk to list os_composite partitions, then dd each into a raw GPT image
# 19 partitions: boot_a/b, init_boot_a/b, metadata, misc, super, userdata,
# vbmeta_a/b, vbmeta_system_a/b, vbmeta_system_dlkm_a/b,
# vbmeta_vendor_dlkm_a/b, vendor_boot_a/b, cuttlefish_example_custom
```

Update `cuttlefish_config.json` to point to `raw_disk.img`.

### 12.5 Create Raw Persistent Disk (FRP Fix — Critical)

> **IMPORTANT**: This step must be performed after EVERY `assemble_cvd` run. Assemble
> overwrites `persistent_composite.img` with a composite_disk protobuf file that crosvm
> cannot parse on aarch64.

The `persistent_composite.img` (composite_disk protobuf) is not parsed by crosvm — the guest
sees it as a single 4MB block with no partitions (no GPT). Without it, `/dev/block/by-name/frp`
is missing, causing `PersistentDataBlockService` to timeout and crash system_server in a loop
(`sys.init.updatable_crashing=1`, boot never completes).

**Fix: Create a raw GPT persistent disk:**

```bash
INST=~/work/aosp_imgs/cuttlefish/instances/cvd-1
RAW_PERSISTENT=${INST}/raw_persistent.img

dd if=/dev/zero of=$RAW_PERSISTENT bs=1M count=4

sudo sgdisk \
  -n 1:2048:+1M   -c 1:frp \
  -n 2:0:+72K     -c 2:bootconfig \
  -n 3:0:+72K     -c 3:uboot_env \
  -n 4:0:+64K     -c 4:persistent_vbmeta \
  $RAW_PERSISTENT

LOOP=$(sudo losetup --show -fP $RAW_PERSISTENT)
sudo dd if=${INST}/internal/factory_reset_protected.img of=${LOOP}p1 bs=4096
sudo dd if=${INST}/internal/bootconfig                  of=${LOOP}p2 bs=4096
sudo dd if=${INST}/uboot_env.img                        of=${LOOP}p3 bs=4096
sudo dd if=${INST}/persistent_vbmeta.img                of=${LOOP}p4 bs=4096
sudo losetup -d $LOOP

# Replace the composite version
cp ${INST}/persistent_composite.img ${INST}/persistent_composite.img.orig
cp $RAW_PERSISTENT ${INST}/persistent_composite.img
```

### 12.6 Apply Image Patches (Same as Standalone)

Apply the same patches as Sections 5-7 to the raw_disk.img:
- Patch vbmeta_a + vbmeta_system_a flags to 0x03 (Section 5)
- Binary-patch vendor_a encryption flags in super (Section 6)
- Format metadata as ext4, userdata as f2fs (Section 7)

### 12.7 Launch

```bash
cd ~/work/aosp_imgs
ulimit -n 65536
ANDROID_HOST_OUT=$PWD HOME=$PWD nohup ./bin/run_cvd > /tmp/run_cvd_out.log 2>&1 &
```

#### 12.7.1 Relaunch VM Without Rebooting Host

If you need to restart the VM without rebooting the Orin Nano (e.g., after a crash,
config change, or to test a fresh boot), follow this procedure:

```bash
# 1. Kill ALL Cuttlefish processes (comprehensive — covers all host daemons and orphans)
sudo pkill -9 -f "run_cvd|crosvm|process_restarter|log_tee|modem_sim|echo_server|secure_env|netsimd|socket_vsock|tcp_connector|adb_connector|tombstone|casimir|gnss_grpc|screen_rec|cf_vhost|operator_proxy|webRTC|sensors_sim|cf_vhost_user_input|control_env|logcat_receiver|kernel_log|cuttlefish_example"
sleep 2

# 2. Verify all vsock ports are free (output should be empty)
sudo ss -lnp | grep v_str

# 3. Clean stale Unix sockets and recreate required directories
find /tmp/cf_avd_1000 -type s -delete 2>/dev/null
mkdir -p /tmp/cf_avd_1000/cvd-1/internal /tmp/cf_avd_1000/cvd-1/grpc_socket

# 4. Relaunch
cd ~/work/aosp_imgs && ulimit -n 65536 && \
  ANDROID_HOST_OUT=$PWD HOME=$PWD \
  nohup ./bin/run_cvd > /tmp/run_cvd_out.log 2>&1 &

# 5. Verify boot (~60s)
sleep 60
adb -s 127.0.0.1:6520 shell getprop sys.boot_completed  # should return 1
```

> **Why the comprehensive pkill?** Orphaned processes (especially `modem_simulator` on
> vsock port 9600, `netsimd` on 7300/7500, `tombstone_receiver` on 6600) hold vsock ports
> open. If not killed, the new run_cvd fails with "Address already in use" on those ports,
> causing cascade daemon failures and boot never completing.
>
> **Weston stays running** — you do NOT need to restart Weston between VM relaunches.
> Only restart Weston if display output stops working.

### 12.8 Boot Result

- `sys.boot_completed=1` at **~60 seconds**
- `VIRTUAL_DEVICE_BOOT_COMPLETED` in run_cvd output
- **0 zygote crashes** (vs 11+ without FRP fix)
- All host daemons running (modem_simulator, netsimd, wmediumd, casimir, sensors, adb proxy, logcat)
- vdb (persistent disk) correctly detected with 4 GPT partitions (frp, bootconfig, uboot_env, persistent_vbmeta)
- BT/WiFi/NFC working via host daemons (netsimd, wmediumd, casimir)
- `vendor.light-cuttlefish` still crashes (no host lights daemon) — harmless with `disable_rescue_party=true`
- `threadnetwork-service` crashes (no Thread radio) — harmless, does not block boot

#### 12.8.1 Complete Post-Assemble Procedure (Checklist)

After every `assemble_cvd`, perform these steps in order:

```bash
# 1. Restore kernel as bootloader (was u-boot during assemble)
cp boot_unpacked/kernel etc/bootloader_aarch64/bootloader.crosvm

# 2. Patch config (Section 12.2.1)
python3 -c "
import json, glob
for f in glob.glob('cuttlefish/**/cuttlefish_config.json', recursive=True):
    with open(f) as fh: cfg = json.load(fh)
    for inst in cfg.get('instances', [cfg]):
        inst['vhost_user_vsock'] = False
        inst['enable_gpu_vhost_user'] = False
    with open(f, 'w') as fh: json.dump(cfg, fh, indent=4)
    print(f'Patched: {f}')
"

# 3. Recreate raw GPT persistent disk (Section 12.5)
INST=cuttlefish/instances/cvd-1
dd if=/dev/zero of=${INST}/persistent_composite.img bs=1M count=4
sudo sgdisk -n 1:2048:+1M -c 1:frp -n 2:0:+72K -c 2:bootconfig \
            -n 3:0:+72K -c 3:uboot_env -n 4:0:+64K -c 4:persistent_vbmeta \
            ${INST}/persistent_composite.img
LOOP=$(sudo losetup --show -fP ${INST}/persistent_composite.img)
sudo dd if=${INST}/internal/factory_reset_protected.img of=${LOOP}p1 bs=4096
sudo dd if=${INST}/internal/bootconfig of=${LOOP}p2 bs=4096
sudo dd if=${INST}/uboot_env.img of=${LOOP}p3 bs=4096
sudo dd if=${INST}/persistent_vbmeta.img of=${LOOP}p4 bs=4096
sudo losetup -d $LOOP

# 4. Patch vbmeta (all 4 files)
for f in vbmeta.img vbmeta_system.img vbmeta_system_dlkm.img vbmeta_vendor_dlkm.img; do
  printf '\x00\x00\x00\x03' | dd of=$f seek=120 bs=1 count=4 conv=notrunc
done

# 5. Ready to launch (Section 12.7)
```

---

## 13. Physical Display + Touch (Weston Fullscreen)

With run_cvd and `enable_gpu_vhost_user=false`, Android renders via gfxstream **inline** in the
main crosvm process (no separate vhost-user-gpu device). The main `bin/crosvm` wrapper redirects
the wayland socket from the internal CF path to the real Weston display on DP output.

### 13.1 GPU Acceleration (Confirmed Working)

run_cvd with `--gpu_mode=gfxstream` uses the Tegra GPU directly:

```
Host GPU: NVIDIA Tegra Orin (nvgpu) / integrated
GLES: 3.2 NVIDIA 540.4.0
SurfaceFlinger: SkiaGL Backend (Ganesh)
host_composition: v1 + v2 enabled
```

No virglrenderer needed — gfxstream speaks directly to the host EGL/GLES.

### 13.2 Load nvidia-drm (Required for DP Output)

The Tegra DRM driver (`card0` = `tegra_drm/host1x`) has NO connectors by default.
You must load `nvidia-drm` with `modeset=1` to get DP output:

```bash
sudo modprobe nvidia-drm modeset=1
# Creates /dev/dri/card0 with connector card0-DP-1
# (card number may vary — check with: ls /sys/class/drm/ | grep DP)
# Verify:
cat /sys/class/drm/card0-DP-1/status
# Should show: connected
```

> **Note**: Must re-run after every reboot. To persist:
> ```bash
> echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
> echo "nvidia-drm" | sudo tee -a /etc/modules-load.d/nvidia-drm.conf
> ```

#### 13.2.1 Swap Render Nodes (Required After Reboot)

When `nvidia-drm` loads at boot, it registers as `card0` with render node `renderD128`.
The tegra host1x driver registers as `card1` with `renderD129`. NVIDIA's Weston GL renderer
and EGL internally require the "tegra" GBM backend, but `renderD128` (nvidia-drm) reports
backend name "nvidia" which both NVIDIA EGL and Mesa reject.

**Root cause**: Module load order after reboot inverts render node numbering.
Ref: [NVIDIA Forum — same issue](https://forums.developer.nvidia.com/t/348933)

**Fix**: Swap the render device node filenames before starting Weston:

```bash
# Check which render node is nvidia-drm vs tegra
for r in /dev/dri/renderD*; do
  drv=$(cat /sys/class/drm/$(basename $r)/device/uevent 2>/dev/null | grep DRIVER | cut -d= -f2)
  echo "$r: $drv"
done

# If renderD128=nvidia-drm (nv_platform), swap them:
sudo mv /dev/dri/renderD128 /dev/dri/renderD128_bak
sudo mv /dev/dri/renderD129 /dev/dri/renderD128
sudo mv /dev/dri/renderD128_bak /dev/dri/renderD129
```

After swapping: renderD128=tegra, renderD129=nvidia-drm. Weston's GL renderer picks
renderD129 (matching card0's device) and EGL initializes with NVIDIA Tegra (GLES 3.2).

> **Auto-detect script** (safe to run even if already correct):
> ```bash
> # Swap only if renderD128 is nvidia-drm (nv_platform)
> if grep -q nv_platform /sys/class/drm/renderD128/device/uevent 2>/dev/null; then
>   sudo mv /dev/dri/renderD128 /dev/dri/renderD128_bak
>   sudo mv /dev/dri/renderD129 /dev/dri/renderD128
>   sudo mv /dev/dri/renderD128_bak /dev/dri/renderD129
>   echo "Render nodes swapped (nvidia-drm → renderD129, tegra → renderD128)"
> else
>   echo "Render nodes already in correct order"
> fi
> ```

#### 13.2.2 EGL Config Shim for gfxstream (Required)

NVIDIA's bare EGL display (no X11/Wayland window system) only exposes `EGL_PBUFFER_BIT`
configs — no `EGL_WINDOW_BIT`. gfxstream's config probe uses default surface type
(`EGL_WINDOW_BIT` per EGL spec) and finds 0 matching configs, failing initialization.

**Fix**: Build and deploy `egl_config_shim.so` — an LD_PRELOAD library that adds
`EGL_SURFACE_TYPE=EGL_PBUFFER_BIT` to config queries missing a surface type:

```bash
cat > ~/work/aosp_imgs/egl_config_shim.c << 'EOF'
#define _GNU_SOURCE
#include <dlfcn.h>
#include <EGL/egl.h>
#include <stdlib.h>
#include <string.h>

EGLBoolean eglChooseConfig(EGLDisplay dpy, const EGLint *attrib_list,
                           EGLConfig *configs, EGLint config_size, EGLint *num_config) {
    static EGLBoolean (*real_fn)(EGLDisplay, const EGLint*, EGLConfig*, EGLint, EGLint*) = NULL;
    if (!real_fn) real_fn = dlsym(RTLD_NEXT, "eglChooseConfig");
    int has_surface_type = 0, len = 0;
    if (attrib_list) {
        for (int i = 0; attrib_list[i] != EGL_NONE; i += 2) {
            if (attrib_list[i] == EGL_SURFACE_TYPE) has_surface_type = 1;
            len = i + 2;
        }
    }
    if (!has_surface_type && attrib_list && len > 0) {
        EGLint *new_attribs = malloc((len + 3) * sizeof(EGLint));
        memcpy(new_attribs, attrib_list, len * sizeof(EGLint));
        new_attribs[len] = EGL_SURFACE_TYPE;
        new_attribs[len + 1] = EGL_PBUFFER_BIT;
        new_attribs[len + 2] = EGL_NONE;
        EGLBoolean ret = real_fn(dpy, new_attribs, configs, config_size, num_config);
        free(new_attribs);
        return ret;
    }
    return real_fn(dpy, attrib_list, configs, config_size, num_config);
}
EOF
gcc -shared -fPIC -o ~/work/aosp_imgs/egl_config_shim.so \
    ~/work/aosp_imgs/egl_config_shim.c -ldl
```

The `bin/crosvm` wrapper (Section 12.3) loads this via `export LD_PRELOAD=${BASE_DIR}/egl_config_shim.so`.

### 13.3 Start Weston (Fullscreen, No Panel)

Use `panel-position=none` so Android gets the full 1280x720 without Weston's taskbar
eating pixels. **Critical**: `idle-time=0` in `[core]` prevents Weston from blanking
the display after its default timeout (which disables the CRTC and kills video signal).

**Step 1: Create persistent weston.ini (one-time setup)**

`/tmp` is cleared on reboot, so store the config in a persistent location:

```bash
cat > ~/work/aosp_imgs/weston.ini << 'EOF'
[core]
idle-time=0

[shell]
panel-position=none
locking=false

[output]
name=DP-1
mode=1280x720
EOF
```

**Step 2: Detect the correct DRM card**

The DP connector can be on `card0` or `card1` depending on module load order.
Auto-detect which card has the DP connector:

```bash
# Find which card has the DP connector
DRM_CARD=$(ls /sys/class/drm/ | grep -o 'card[0-9]*-DP' | head -1 | cut -d- -f1)
echo "DP connector on: $DRM_CARD"   # e.g. "card0"

# Verify it's connected
cat /sys/class/drm/${DRM_CARD}-DP-1/status   # should show: connected
```

**Step 3: Swap render nodes + Start Weston**

```bash
# Swap render nodes if needed (Section 13.2.1)
if grep -q nv_platform /sys/class/drm/renderD128/device/uevent 2>/dev/null; then
  sudo mv /dev/dri/renderD128 /dev/dri/renderD128_bak
  sudo mv /dev/dri/renderD129 /dev/dri/renderD128
  sudo mv /dev/dri/renderD128_bak /dev/dri/renderD129
  echo "Render nodes swapped"
fi

# Auto-detect card and launch with GL renderer
DRM_CARD=$(ls /sys/class/drm/ | grep -o 'card[0-9]*-DP' | head -1 | cut -d- -f1)
mkdir -p /tmp/weston-run && chmod 700 /tmp/weston-run
export XDG_RUNTIME_DIR=/tmp/weston-run
export SEATD_VTBOUND=0
weston --backend=drm --drm-device=$DRM_CARD --renderer=gl \
       --config=$HOME/work/aosp_imgs/weston.ini \
       --continue-without-input --log=/tmp/weston.log 2>/dev/null &
sleep 4

# Make socket writable by nano user (crosvm GPU runs as nano)
sudo chmod 777 /tmp/weston-run /tmp/weston-run/wayland-0

# Verify GL renderer (not pixman!)
grep "EGL vendor\|GL version\|DP-1.*enabled" /tmp/weston.log
# Should show: EGL vendor: NVIDIA, GL version: OpenGL ES 3.2 NVIDIA 540.4.0
ls /tmp/weston-run/wayland-0  # socket exists
```

> **Why `--renderer=gl`?** Without it, Weston may fall back to pixman (CPU renderer)
> which doesn't advertise `zwp_linux_dmabuf_v1` protocol needed for zero-copy GPU frame sharing.

### 13.4 GPU Device Wrapper (`bin/prebuilts/crosvm`) — NOT NEEDED

> **This section is only relevant when `enable_gpu_vhost_user=true` (separate GPU process).**
> With the recommended `enable_gpu_vhost_user=false` configuration (Section 12.2.1), the GPU
> renders inline in the main crosvm process. The `bin/crosvm` wrapper (Section 12.3) handles
> wayland redirection and display dimension swapping directly. Skip this section.

<details>
<summary>Legacy: GPU device wrapper for vhost-user-gpu mode (click to expand)</summary>

run_cvd launches the GPU device as a separate process:
```
prebuilts/crosvm device gpu --wayland-sock=/tmp/cf_avd_1000/cvd-1/internal/frames.sock ...
```

We need to redirect this to Weston's socket AND swap the display to landscape.
Create a wrapper:

```bash
# Backup original
mv ~/work/aosp_imgs/bin/prebuilts/crosvm ~/work/aosp_imgs/bin/prebuilts/crosvm.real

# Create wrapper
cat > ~/work/aosp_imgs/bin/prebuilts/crosvm << 'WRAPPER'
#!/bin/bash
REAL="$(dirname "$(readlink -f "$0")")/crosvm.real"
ARGS=()
for arg in "$@"; do
    if [[ "$arg" == "--wayland-sock=/tmp/cf_avd_1000/cvd-1/internal/frames.sock" ]]; then
        ARGS+=("--wayland-sock=/tmp/weston-run/wayland-0")
    elif [[ "$arg" == *'"windowed":[720,1280]'* ]]; then
        ARGS+=("${arg//\"windowed\":\[720,1280\]/\"windowed\":[1280,720]}")
    else
        ARGS+=("$arg")
    fi
done
export XDG_RUNTIME_DIR=/tmp/weston-run
exec "$REAL" "${ARGS[@]}"
WRAPPER
chmod +x ~/work/aosp_imgs/bin/prebuilts/crosvm
```

This wrapper does two things:
1. Redirects GPU output from internal `frames.sock` → Weston's `wayland-0`
2. Swaps display from portrait (720×1280) to landscape (1280×720) matching the monitor

</details>

### 13.5 Main crosvm Wrapper with Touch Input (`bin/crosvm`)

The wrapper is identical to Section 12.3 above (same KCMDLINE, same touch auto-detect,
same persistent INITRD path). See Section 12.3 for the full script.

> **Touch auto-detection**: The wrapper uses `grep -l eGalax /sys/class/input/event*/device/name`
> to find the correct event device at launch time. This handles device renumbering across reboots
> (event5 → event1, etc.). The device must be readable (`chmod 666`) — see Section 13.6.

### 13.6 Touch Device Setup

The Lilliput monitor's USB touch (eGalax EXC3147, vendor=0eef product=c000) appears as
`/dev/input/event5`. Must be readable by the user running crosvm:

```bash
sudo chmod 666 /dev/input/event5

# To persist across reboots, create a udev rule:
echo 'SUBSYSTEM=="input", ATTRS{idVendor}=="0eef", ATTRS{idProduct}=="c000", MODE="0666"' | \
  sudo tee /etc/udev/rules.d/99-egalax-touch.rules
sudo udevadm control --reload-rules
```

### 13.7 Full Restart Procedure

```bash
# 1. Kill ALL CF processes (including orphans on vsock ports)
sudo pkill -9 -f "run_cvd|crosvm|process_restarter|log_tee|modem_sim|echo_server|secure_env|sensors_sim|webRTC|operator|tombstone|netsimd|socket_vsock|tcp_connector|adb_connector|casimir|gnss_grpc|screen_rec|cf_vhost"
sudo pkill -9 weston
sleep 2

# 2. Verify all vsock ports free (should be empty)
sudo ss -lnp | grep v_str

# 3. Clean sockets and ensure required directories exist
find /tmp/cf_avd_1000 -type s -delete 2>/dev/null
mkdir -p /tmp/cf_avd_1000/cvd-1/internal /tmp/cf_avd_1000/cvd-1/grpc_socket
rm -f /tmp/run_cvd_out.log

# 4. Swap render nodes if needed (Section 13.2.1)
if grep -q nv_platform /sys/class/drm/renderD128/device/uevent 2>/dev/null; then
  sudo mv /dev/dri/renderD128 /dev/dri/renderD128_bak
  sudo mv /dev/dri/renderD129 /dev/dri/renderD128
  sudo mv /dev/dri/renderD128_bak /dev/dri/renderD129
  echo "Render nodes swapped"
fi

# 5. Restart Weston with GL renderer (uses persistent config + auto-detects DRM card)
DRM_CARD=$(ls /sys/class/drm/ | grep -o 'card[0-9]*-DP' | head -1 | cut -d- -f1)
mkdir -p /tmp/weston-run && chmod 700 /tmp/weston-run
export XDG_RUNTIME_DIR=/tmp/weston-run SEATD_VTBOUND=0
weston --backend=drm --drm-device=$DRM_CARD --renderer=gl \
  --config=/home/nano/work/aosp_imgs/weston.ini \
  --continue-without-input --log=/tmp/weston.log 2>/dev/null &
sleep 4
# CRITICAL: GPU device (runs as nano) needs write access to wayland socket
sudo chmod 777 /tmp/weston-run /tmp/weston-run/wayland-0

# 6. Verify Weston GL renderer and touch accessible
grep "EGL vendor" /tmp/weston.log  # should show: NVIDIA
ls /tmp/weston-run/wayland-0       # socket exists

# 7. Launch run_cvd
cd /home/nano/work/aosp_imgs && ulimit -n 65536 && \
  ANDROID_HOST_OUT=/home/nano/work/aosp_imgs \
  HOME=/home/nano/work/aosp_imgs \
  XDG_RUNTIME_DIR=/tmp/weston-run \
  nohup ./bin/run_cvd > /tmp/run_cvd_out.log 2>&1 &

# 8. Wait ~60s then verify
sleep 60
adb -s 127.0.0.1:6520 shell getprop sys.boot_completed  # should return 1

# 9. Set density for proper launcher layout on 1280x720
adb shell wm density 160   # fits nav bar + launcher icons comfortably
```

### 13.8 Persistence Across Reboots

These configs ensure the Nano is ready to launch run_cvd immediately after a reboot
(no manual setup required except starting Weston and launching run_cvd):

| File | Purpose | Created by |
|------|---------|-----------|
| `/etc/modprobe.d/nvidia-drm.conf` | `options nvidia-drm modeset=1` — DP output available at boot | `echo "options nvidia-drm modeset=1" \| sudo tee ...` |
| `/etc/modules-load.d/nvidia-drm.conf` | Auto-loads nvidia-drm module | `echo "nvidia-drm" \| sudo tee ...` |
| `/etc/systemd/system/cuttlefish-tap.service` | Creates cvd-{m,e,w}tap-01 tap devices on boot | systemd oneshot service |
| `/etc/tmpfiles.d/cuttlefish.conf` | Creates `/tmp/cf_avd_1000/...`, `/tmp/cf_env_1000/...`, `/run/cuttlefish/operator` | systemd-tmpfiles |
| `/etc/udev/rules.d/99-egalax-touch.rules` | `MODE="0666"` for eGalax touch — ensures wrapper can pass it to crosvm | udev rule |
| `~/work/aosp_imgs/weston.ini` | Weston config (idle-time=0, no panel, DP-1 1280x720) — survives reboot unlike `/tmp` | Section 13.3 one-time setup |
| `~/work/aosp_imgs/egl_config_shim.so` | LD_PRELOAD shim fixing gfxstream GLES config detection on NVIDIA bare EGL | Section 13.2.2 (one-time build) |
| `/etc/modules-load.d/vsock.conf` | Loads vsock/vhost modules at boot | Section 9.5 |

> **Non-persistent (must redo after every reboot)**:
> - Render node swap (`/dev/dri/renderD128` ↔ `renderD129`) — Section 13.2.1

After reboot, only these commands are needed (copy-paste ready):

```bash
# 1. Swap render nodes if needed (Section 13.2.1)
if grep -q nv_platform /sys/class/drm/renderD128/device/uevent 2>/dev/null; then
  sudo mv /dev/dri/renderD128 /dev/dri/renderD128_bak
  sudo mv /dev/dri/renderD129 /dev/dri/renderD128
  sudo mv /dev/dri/renderD128_bak /dev/dri/renderD129
  echo "Render nodes swapped"
fi

# 2. Start Weston with GL renderer (auto-detects DRM card, uses persistent config)
DRM_CARD=$(ls /sys/class/drm/ | grep -o 'card[0-9]*-DP' | head -1 | cut -d- -f1)
mkdir -p /tmp/weston-run && chmod 700 /tmp/weston-run
export XDG_RUNTIME_DIR=/tmp/weston-run SEATD_VTBOUND=0
weston --backend=drm --drm-device=$DRM_CARD --renderer=gl \
       --config=/home/nano/work/aosp_imgs/weston.ini \
       --continue-without-input --log=/tmp/weston.log 2>/dev/null &
sleep 4
sudo chmod 777 /tmp/weston-run /tmp/weston-run/wayland-0

# 3. Launch run_cvd
cd ~/work/aosp_imgs && ulimit -n 65536 && \
  ANDROID_HOST_OUT=$PWD HOME=$PWD XDG_RUNTIME_DIR=/tmp/weston-run \
  nohup ./bin/run_cvd > /tmp/run_cvd_out.log 2>&1 &

# 4. Verify (~60s)
sleep 60 && adb -s 127.0.0.1:6520 shell getprop sys.boot_completed
```

> **No pkill/cleanup needed after a fresh reboot** — nothing is running yet.
> The `weston.ini` is stored persistently at `~/work/aosp_imgs/weston.ini` (not in `/tmp`).
> The render node swap is NOT persistent — must be done after every reboot.

### 13.9 Known Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Port 6600/7300/7500 "Address already in use" | Orphaned `tombstone_receiver`, `netsimd`, `socket_vsock_proxy` from previous run holding **vsock** ports (not TCP) | Kill with `sudo pkill -9 -f tombstone\|netsimd\|socket_vsock`; verify with `sudo ss -lnp \| grep v_str` |
| `netsimd` exits → cascade shutdown | vsock port 7300/7500 held by orphan | Same as above — ensure ALL orphans killed before relaunch |
| Top/bottom of screen clipped | Weston's desktop-shell adds a top panel | Use `panel-position=none` in weston.ini (Section 13.3) |
| Android stuck in portrait (720×1280) on landscape monitor | GPU device configured for phone portrait mode | prebuilts/crosvm wrapper swaps to 1280×720 (Section 13.4) |
| Touch offset/misaligned | Touch calibration doesn't match display transform | May need `weston.ini` `[output]` transform or Android `idc` file |
| Screen blanks after ~5min | Weston idle timer disables CRTC (DRM `enabled`=disabled) | Add `idle-time=0` under `[core]` in weston.ini (Section 13.3) |
| GPU renders but black screen | wayland-0 socket owned by root, nano user can't write | `sudo chmod 777 /tmp/weston-run/wayland-0` after Weston starts |
| `XDG_RUNTIME_DIR not set` fatal | Weston launched without env var | Always use `sudo bash -c 'XDG_RUNTIME_DIR=/tmp/weston-run weston ...'` |
| run_cvd exit 127 | Wrong working directory (must be `~/work/aosp_imgs`) | `cd ~/work/aosp_imgs` before launching |
| VM reboot loop (165+ reboots, `sys.boot_completed` empty) | `vendor.light-cuttlefish` HAL crashes → rescue party triggers `reboot_on_failure`. The light HAL needs `vsock_lights_port/cid` pointing to a host daemon that doesn't exist | Add `androidboot.disable_rescue_party=true` to KCMDLINE. Boot completes at ~26s; light HAL crashes harmlessly at ~226s without rebooting |
| Touch not working after boot | `/dev/input/event*` permissions are `crw-rw----` (root:input). Wrapper's `[ -r ]` check fails at launch → touch arg not passed | Create udev rule: `SUBSYSTEM=="input", ATTRS{idVendor}=="0eef", MODE="0666"` in `/etc/udev/rules.d/99-egalax-touch.rules`. Must be set BEFORE run_cvd launch |
wm density 160

### 13.10 Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════════╗
║                    NVIDIA Jetson Orin Nano — Android 17 Cuttlefish KVM                  ║
║                     p3768-0000+p3767-0005  •  ARM64  •  8 GB RAM                        ║
╠══════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                         ║
║  ┌─ HARDWARE ──────────────────────────────────────────────────────────────────────────┐ ║
║  │                                                                                     │ ║
║  │   ┌──────────────┐    ┌──────────────────┐    ┌───────────────┐    ┌─────────────┐  │ ║
║  │   │  Tegra Orin  │    │  Lilliput 1280×  │    │  eGalax USB   │    │  Ethernet   │  │ ║
║  │   │  nvgpu       │    │  720 Monitor     │    │  Touch Panel  │    │  NIC (eth0) │  │ ║
║  │   │  GLES 3.2    │    │  (DP→HDMI)       │    │  EXC3147-27xx │    │             │  │ ║
║  │   └──────┬───────┘    └────────┬─────────┘    └──────┬────────┘    └──────┬──────┘  │ ║
║  │          │ DRM                 │ HDMI                │ USB HID           │          │ ║
║  └──────────┼─────────────────────┼─────────────────────┼───────────────────┼──────────┘ ║
║             │                     │                     │                   │            ║
║  ┌─ HOST OS LAYER (L4T R36.4.7 · Ubuntu 22.04 · Kernel 5.15.148-tegra) ───────────────┐ ║
║  │                                                                                     │ ║
║  │   ┌──────────────────┐   ┌─────────────────┐   ┌──────────────┐   ┌──────────────┐ │ ║
║  │   │  nvidia-drm      │   │  KVM / vhost    │   │  evdev       │   │  virtio-net  │ │ ║
║  │   │  modeset=1       │   │  /dev/kvm        │   │  /dev/input/ │   │  TAP devices │ │ ║
║  │   │  /dev/dri/card1  │   │  Hardware Virt.  │   │  event1      │   │  cvd-*tap-01 │ │ ║
║  │   └────────┬─────────┘   └────────┬────────┘   └──────┬───────┘   └──────┬───────┘ │ ║
║  │            │                      │                    │                  │          │ ║
║  └────────────┼──────────────────────┼────────────────────┼──────────────────┼──────────┘ ║
║               │                      │                    │                  │            ║
║  ┌─ HOST USERSPACE ────────────────────────────────────────────────────────────────────┐  ║
║  │                                                                                     │ ║
║  │   ┌───────────────────────────────┐        ┌────────────────────────────────────┐   │ ║
║  │   │       WESTON COMPOSITOR       │        │        run_cvd ORCHESTRATOR        │   │ ║
║  │   │   nvidia-l4t-weston 13.0.0    │        │    ~/work/aosp_imgs/bin/run_cvd    │   │ ║
║  │   │                               │        │                                    │   │ ║
║  │   │  Backend:  drm (card1-DP-1)   │        │  Manages:                          │   │ ║
║  │   │  Socket:   /tmp/weston-run/   │        │    • VM lifecycle (crosvm)          │   │ ║
║  │   │            wayland-0          │        │    • Host daemon processes          │   │ ║
║  │   │  Mode:     1280×720 @ 60Hz    │        │    • Networking & vsock channels    │   │ ║
║  │   │  Config:   idle-time=0        │        │    • Log collection                 │   │ ║
║  │   │            panel-position=none│        │                                    │   │ ║
║  │   └───────────┬───────────────────┘        └──────────┬─────────────────────────┘   │ ║
║  │               │                                       │                             │ ║
║  │               │ Wayland                               │ spawns & manages             │ ║
║  │               │ Protocol                    ┌─────────┴──────────────────────┐      │ ║
║  │               │                             │                                │      │ ║
║  │   ┌───────────▼───────────────────┐   ┌─────▼──────────────────────────────┐ │      │ ║
║  │   │   vhost-user-gpu PROCESS      │   │      HOST DAEMON PROCESSES         │ │      │ ║
║  │   │   (prebuilts/crosvm device)   │   │                                    │ │      │ ║
║  │   │                               │   │  modem_simulator                   │ │      │ ║
║  │   │  Renders Android frames to    │   │  gnss_grpc_proxy                   │ │      │ ║
║  │   │  Weston via Wayland client    │   │  secure_env (Keymint/Gatekeeper)   │ │      │ ║
║  │   │                               │   │  log_tee (kernel/logcat pipes)     │ │      │ ║
║  │   │  --wayland-sock wayland-0     │   │  metrics_cf                        │ │      │ ║
║  │   │  --params {windowed:          │   │  config_server                     │ │      │ ║
║  │   │    [1280,720]}                │   │  tombstone_receiver                │ │      │ ║
║  │   │                               │   │  adb_connector → vsock:3:5555     │ │      │ ║
║  │   │  XDG_RUNTIME_DIR=            │   │  webRTC / operator (control UI)    │ │      │ ║
║  │   │    /tmp/weston-run            │   │                                    │ │      │ ║
║  │   └───────────┬───────────────────┘   └────────────────┬───────────────────┘ │      │ ║
║  │               │ vhost-user socket                      │ vsock / pipes       │      │ ║
║  │               │ (gpu.sock)                             │                     │      │ ║
║  └───────────────┼────────────────────────────────────────┼─────────────────────┼──────┘  ║
║                  │                                        │                     │         ║
║  ╔═══════════════╧════════════════════════════════════════╧═════════════════════╧═══════╗ ║
║  ║                          crosvm VIRTUAL MACHINE (KVM)                                ║ ║
║  ║                         bin/crosvm wrapper → direct kernel boot                      ║ ║
║  ║                                                                                      ║ ║
║  ║   ┌─ VIRTIO DEVICE BUS ────────────────────────────────────────────────────────────┐ ║ ║
║  ║   │  virtio-gpu ←── vhost-user-gpu    (gfxstream rendering pipeline)               │ ║ ║
║  ║   │  virtio-net ←── TAP cvd-mtap-01   (mobile data)                                │ ║ ║
║  ║   │  virtio-net ←── TAP cvd-etap-01   (ethernet bridge)                            │ ║ ║
║  ║   │  virtio-net ←── TAP cvd-wtap-01   (WiFi emulation)                             │ ║ ║
║  ║   │  virtio-input ←── evdev passthru  (/dev/input/event1 → eGalax touch)           │ ║ ║
║  ║   │  virtio-blk ←── super.img, userdata.img, vendor_boot.img, etc.                 │ ║ ║
║  ║   │  vsock CID=3 ←── host daemon communication channels                            │ ║ ║
║  ║   └────────────────────────────────────────────────────────────────────────────────┘ ║ ║
║  ║                                                                                      ║ ║
║  ║   ┌─ GUEST KERNEL (6.12.74-android16-6-gf44d901e8100-ab13097027) ─────────────────┐ ║ ║
║  ║   │                                                                                 │ ║ ║
║  ║   │  Boot:    vmlinuz + initrd_with_bootconfig.img (persistent, 21 MB)              │ ║ ║
║  ║   │  KCMDLINE:  vendor.apex properties (4x) · disable_rescue_party=true             │ ║ ║
║  ║   │             vsock_lights_port=6900 · vsock_lights_cid=3                         │ ║ ║
║  ║   │  Modules:   virtio_gpu · virtio_net · virtio_input · virtio_blk                 │ ║ ║
║  ║   └────────────────────────────────────────────────────────────────────────────────┘ ║ ║
║  ║                                                                                      ║ ║
║  ║   ┌─ ANDROID 17 CUTTLEFISH (aosp_cf_arm64_only_phone-trunk_staging-userdebug) ────┐ ║ ║
║  ║   │                                                                                 │ ║ ║
║  ║   │   ┌─────────────────────────────── GRAPHICS PIPELINE ────────────────────────┐  │ ║ ║
║  ║   │   │                                                                          │  │ ║ ║
║  ║   │   │  Android App  →  SurfaceFlinger  →  HWComposer (HWC3)                   │  │ ║ ║
║  ║   │   │       │                                    │                              │  │ ║ ║
║  ║   │   │       ▼                                    ▼                              │  │ ║ ║
║  ║   │   │  EGL/GLES (gfxstream)  ──────→  virtio-gpu  ──→  vhost-user  ──→  HOST  │  │ ║ ║
║  ║   │   │  (libEGL_emulation.so)          (DRM render)     (gpu.sock)      GPU     │  │ ║ ║
║  ║   │   │                                                                          │  │ ║ ║
║  ║   │   └──────────────────────────────────────────────────────────────────────────┘  │ ║ ║
║  ║   │                                                                                 │ ║ ║
║  ║   │   ┌───────────────────────── KEY ANDROID SERVICES ───────────────────────────┐  │ ║ ║
║  ║   │   │                                                                          │  │ ║ ║
║  ║   │   │  init (1st/2nd stage)  →  apexd (bootstrap + default)  →  zygote        │  │ ║ ║
║  ║   │   │  system_server  →  PackageManager  →  Launcher                           │  │ ║ ║
║  ║   │   │  InputManagerService  ←  /dev/input/event* (virtio-input touch)          │  │ ║ ║
║  ║   │   │  adbd  ←  vsock:2:5555  ←  adb_connector (host)                         │  │ ║ ║
║  ║   │   │                                                                          │  │ ║ ║
║  ║   │   └──────────────────────────────────────────────────────────────────────────┘  │ ║ ║
║  ║   │                                                                                 │ ║ ║
║  ║   │   ┌──────────────────────── CUTTLEFISH HAL LAYER ────────────────────────────┐  │ ║ ║
║  ║   │   │                                                                          │  │ ║ ║
║  ║   │   │  vendor.light-cuttlefish  (vsock → port 6900, crashes harmlessly)        │  │ ║ ║
║  ║   │   │  vendor.sensors-cuttlefish  (vsock → host gnss/sensors)                  │  │ ║ ║
║  ║   │   │  vendor.keymint-cuttlefish  (vsock → host secure_env)                    │  │ ║ ║
║  ║   │   │  vendor.gatekeeper-cuttlefish  (vsock → host secure_env)                 │  │ ║ ║
║  ║   │   │  android.hardware.graphics.composer@3  (HWC3 → virtio-gpu)              │  │ ║ ║
║  ║   │   │                                                                          │  │ ║ ║
║  ║   │   └──────────────────────────────────────────────────────────────────────────┘  │ ║ ║
║  ║   │                                                                                 │ ║ ║
║  ║   │   Boot Timeline:  init → apexd → zygote → system_server → boot_completed      │ ║ ║
║  ║   │                   ~0s     ~3s     ~8s       ~15s            ~26s                │ ║ ║
║  ║   │                                                                                 │ ║ ║
║  ║   └────────────────────────────────────────────────────────────────────────────────┘ ║ ║
║  ║                                                                                      ║ ║
║  ╚══════════════════════════════════════════════════════════════════════════════════════╝ ║
║                                                                                         ║
║  ┌─ DATA FLOW LEGEND ─────────────────────────────────────────────────────────────────┐  ║
║  │                                                                                     │ ║
║  │   ══►  GPU Render Path:   App → gfxstream → virtio-gpu → vhost-user → Weston → DP │ ║
║  │   ──►  Touch Input Path:  eGalax USB → evdev → crosvm --input → virtio-input      │ ║
║  │   ──►  Network Path:      Android → virtio-net → TAP → host bridge → Internet      │ ║
║  │   ──►  ADB Path:          adb (host:6520) → adb_connector → vsock:3:5555 → adbd   │ ║
║  │   ──►  HAL Comm Path:     Cuttlefish HAL → vsock CID=3 → host daemon process       │ ║
║  │                                                                                     │ ║
║  └─────────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                         ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝
```
