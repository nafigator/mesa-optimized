# Mesa x86-64-v4 for Devuan (stable)

Optimized Mesa (video driver) packages for Devuan Excalibur, built with `-march=x86-64-v4 -O3`.

Packages are built automatically via GitHub Actions inside a clean `devuan/devuan:excalibur` Docker container and published in the [Releases](../../releases) section.

## Requirements

- **Distribution:** Devuan Excalibur (stable)
- **Architecture:** amd64
- **CPU:** with **AVX-512** support (x86-64-v4)
  - AMD Zen 4 (Ryzen 7000/8000/9000)
- **GPU:** AMD (RDNA 1/2/3/4) — tested on Radeon 780M (Phoenix)

**Packages will not run** on CPUs without AVX-512. Check support:

```bash
grep -o 'avx512[a-z]*' /proc/cpuinfo | sort -u | head
```

If the output is empty — **do not install these packages**.

## Packages

| Package | Purpose |
|---|---|
| `mesa-libgallium` | Main Gallium library (radeonsi, RADV) |
| `mesa-vulkan-drivers` | Vulkan drivers (RADV for AMD) |
| `libgl1-mesa-dri` | DRI drivers for OpenGL |
| `libglx-mesa0` | GLX library |
| `libegl-mesa0` | EGL library |
| `libgbm1` | GBM (buffers for Wayland/KMS) |
| `libosmesa6` | Offscreen rendering |
| `libxatracker2` | XA tracker (XvBA) |
| `mesa-va-drivers` | VA-API (hardware video decoding) |
| `mesa-vdpau-drivers` | VDPAU (hardware video decoding) |

Debug packages (`*-dbgsym`) are **not included** in releases — they do not affect performance and only take up space.

## Installation

### 1. Download packages from the latest release

```bash
mkdir -p ~/mesa-opt && cd ~/mesa-opt
gh release download --repo nafigator/mesa-optimized --pattern '*.deb'
```

Or download the `.deb` files manually from the [Releases](../../releases) page.
You can exclude `*dev*` packages if you don't need them.

### 2. Back up current packages

```bash
sudo mkdir -p /root/mesa-backup
sudo cp /var/cache/apt/archives/mesa-*.deb /root/mesa-backup/ 2>/dev/null || true
```

### 3. Install

```bash
cd ~/mesa-opt
sudo apt install ./*.deb
```

The `./` prefix is required — otherwise `apt` will look for packages in repositories.

`apt` may mark some packages as `DOWNGRADING` even though the version is the same. This is a replacement of the stock build with a local one, not an actual downgrade.

### 4. Hold versions

To prevent `apt upgrade` from reverting to stock packages:

```bash
sudo apt-mark hold \
  mesa-libgallium mesa-vulkan-drivers libgl1-mesa-dri \
  libglx-mesa0 libegl-mesa0 libgbm1 libosmesa6 \
  libxatracker2 mesa-va-drivers mesa-vdpau-drivers
```

### 5. Reboot

```bash
sudo reboot
```

## Verification

```bash
# OpenGL
glxinfo | grep "OpenGL renderer"

# Vulkan
vulkaninfo --summary | grep -A 3 GPU0
```

Expected output (for Radeon 780M):

```
OpenGL renderer string: AMD Radeon 780M (radeonsi, phoenix, ...)
deviceName = AMD Radeon 780M (RADV PHOENIX)
driverName = radv
driverInfo = Mesa 25.0.7-2+deb13u1
```

## Rollback

If graphics become unstable, freeze, or show artifacts:

```bash
# Unhold
sudo apt-mark unhold \
  mesa-libgallium mesa-vulkan-drivers libgl1-mesa-dri \
  libglx-mesa0 libegl-mesa0 libgbm1 libosmesa6 \
  libxatracker2 mesa-va-drivers mesa-vdpau-drivers

# Reinstall stock versions
sudo apt install --reinstall \
  mesa-libgallium mesa-vulkan-drivers libgl1-mesa-dri \
  libglx-mesa0 libegl-mesa0 libgbm1

sudo reboot
```

## Expected Performance

| Component | Gain | Comment |
|---|---|---|
| CPU part of driver (radeonsi/RADV) | 1–3% | Noticeable only in CPU-bound scenarios |
| Shader compilation (ACO) | **0%** | ACO does not use Mesa build flags |
| Games on iGPU (Radeon 780M) | ~0–2% | Bottleneck is memory bandwidth |

**Honest warning:** do not expect a "magic" speedup. The main benefit of these packages is not FPS but more efficient CPU usage in scenarios where the driver is CPU-bound (high FPS, many draw calls). For Radeon 780M the main limiter is memory, not driver code.

## How It Is Built

GitHub Actions workflow:

1. Starts the `devuan/devuan:excalibur` container.
2. Installs `build-essential`, `devscripts`, `equivs`, `quilt`.
3. Downloads `mesa-vulkan-drivers` source via `apt source`.
4. Installs build dependencies via `mk-build-deps`.
5. Applies Debian patches via `quilt push -a`.
6. Adds `-Dc_args="-march=x86-64-v4 -mtune=znver4 -O3 -fno-plt -fomit-frame-pointer -falign-functions=32 -falign-loops=32 -falign-jumps=32"` and `-Dcpp_args="-march=x86-64-v4 -mtune=znver4 -O3 -fno-plt -fomit-frame-pointer -falign-functions=32 -falign-loops=32 -falign-jumps=32"` to `debian/rules`.
7. Builds packages via `dpkg-buildpackage -b -us -uc`.
8. Uploads `.deb` files as artifacts and to the release.

Source workflow: [`.github/workflows/build-mesa.yml`](.github/workflows/build-mesa.yml).

## Important

- Packages are built **only for Devuan Excalibur**. Installing on Daedalus (oldstable) or other releases may break graphics.
- Packages are **not signed**. Verify integrity using SHA-256 from the release description.
- **Do not install** these packages if you are unsure about AVX-512 support on your CPU.
- The author is not responsible for any system issues. Always have a Live USB ready for recovery.

## License

The build scripts and GitHub Actions workflows in this repository
are licensed under the MIT License. See LICENSE file.

The Mesa source code and Debian packaging files are distributed
under their respective licenses — see the mesa source package
for details. The compiled .deb packages in Releases are
redistributions of Mesa under its original license.
