# AMD ROCm + llama.cpp Setup Guide
## Ubuntu 22.04 LTS — Radeon RX 7900 XTX (gfx1100 / RDNA3)

Tested on: Ubuntu 22.04.5 LTS, kernel 6.8.0 (HWE), ROCm 7.2.4, llama.cpp (CMake build)

---

## 1. Prerequisites & OS Setup

### Upgrade to HWE Kernel (required for RDNA3 support)
The default Ubuntu 22.04 kernel (5.15) has incomplete support for the RX 7900 XTX.
Kernel 6.1+ is required; the HWE kernel (6.8) is recommended.

```bash
sudo apt update
sudo apt install linux-image-generic-hwe-22.04
sudo update-grub
```

### Install GCC 12 Dev Libraries (required for ROCm clang)
```bash
sudo apt install libstdc++-12-dev g++-12
```

### Remove Nvidia Leftovers (if migrating from Nvidia GPU)
```bash
sudo apt purge nvidia-* libnvidia-* -y
sudo apt autoremove -y
```

### Reboot into the new kernel
```bash
sudo reboot
```

After reboot, verify the correct kernel is running:
```bash
uname -r   # should show 6.8.x or newer
```

---

## 2. Verify GPU is Detected

```bash
lspci | grep -i vga
```

Expected output for 7900 XTX:
```
03:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Device 744c (rev c8)
```

Check if the `amdgpu` driver has loaded:
```bash
ls /dev/dri/
```

Expected: `card0` and `renderD128`. If empty, the driver isn't binding — check `sudo dmesg | grep amdgpu`.

---

## 3. Install ROCm

### Download the AMD GPU Installer
Use the **stable ROCm 7.2.4** release (amdgpu-install `30.30.4`).

> **Important:** Do NOT use ROCm 7.13 pre-release — its hipBLAS development headers are
> incomplete and will cause build failures in llama.cpp.

```bash
wget https://repo.radeon.com/amdgpu-install/30.30.4/ubuntu/jammy/amdgpu-install_7.2.4.70204-1_all.deb \
  -O /tmp/amdgpu-install.deb

sudo apt install /tmp/amdgpu-install.deb
sudo apt update
```

### Install ROCm with HIP Libraries SDK
The `hiplibsdk` usecase installs HIP runtimes, ROCm math libraries (including hipBLAS and
rocBLAS), and all development headers needed to build GPU-accelerated applications.

```bash
sudo amdgpu-install --usecase=hiplibsdk --no-32 -y
```

### Add User to GPU Groups
```bash
sudo usermod -aG render,video $USER
```

### Reboot
```bash
sudo reboot
```

### Verify ROCm Installation
```bash
ls /dev/dri/          # card0 and renderD128 must be present
rocminfo              # should list gfx1100 device
```

Expected in `rocminfo` output:
```
Name:                    gfx1100
```

---

## 4. Set Up Environment Variables

Add to `~/.bashrc` (makes settings persistent across sessions):

```bash
export PATH=/opt/rocm-7.2.4/bin:$PATH
export LD_LIBRARY_PATH=/opt/rocm-7.2.4/lib:$LD_LIBRARY_PATH
```

Apply immediately:
```bash
source ~/.bashrc
```

---

## 5. Build llama.cpp with ROCm

### Clone the Repository
```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
```

### Configure with CMake
Key flags:
- `-DGGML_HIP=ON` — enables ROCm/HIP GPU backend
- `-DAMDGPU_TARGETS=gfx1100` — targets RX 7900 XTX specifically (speeds up compilation significantly vs. compiling for all AMD architectures)
- `-DCMAKE_HIP_FLAGS="--gcc-install-dir=..."` — tells ROCm clang where GCC 12 stdlib headers are
- `-DCMAKE_PREFIX_PATH="/opt/rocm-7.2.4"` — points to the versioned ROCm install (not the `/opt/rocm` symlink, which may point to a different version)

```bash
cmake -B build \
  -DGGML_HIP=ON \
  -DAMDGPU_TARGETS=gfx1100 \
  -DCMAKE_HIP_FLAGS="--gcc-install-dir=/usr/lib/gcc/x86_64-linux-gnu/12" \
  -DCMAKE_PREFIX_PATH="/opt/rocm-7.2.4"
```

### Build
```bash
cmake --build build --config Release -j$(nproc)
```

Compilation takes 10–20 minutes. Binaries are placed in `build/bin/`.

---

## 6. Run a Model

```bash
./build/bin/llama-cli -m /path/to/model.gguf -ngl 99 -p "Hello, how are you?"
```

Key flags:
- `-ngl 99` — offloads all 99 layers to VRAM (the 7900 XTX has 24 GB, enough for most models)
- `-m` — path to your `.gguf` model file

Confirm GPU offload is active by looking for this line in startup output:
```
ggml_cuda_init: found 1 ROCm devices
```

---

## GPU Architecture Reference

| GPU           | Architecture | Target   | VRAM  |
|---------------|-------------|----------|-------|
| RX 7900 XTX   | RDNA 3      | gfx1100  | 24 GB |
| RX 7900 XT    | RDNA 3      | gfx1100  | 20 GB |
| RX 7800 XT    | RDNA 3      | gfx1101  | 16 GB |
| RX 7600       | RDNA 3      | gfx1102  | 8 GB  |
| RX 6900 XT    | RDNA 2      | gfx1030  | 16 GB |
| RX 6800 XT    | RDNA 2      | gfx1030  | 16 GB |

For a different GPU, replace `gfx1100` with the correct target from the table above.

---

## Troubleshooting

### `/dev/dri/` is empty after reboot
The `amdgpu` kernel module isn't loading. Check:
```bash
sudo dmesg | grep amdgpu
lspci | grep -i vga   # confirm GPU is detected on PCIe
```
Common causes: kernel too old (upgrade to HWE), Nvidia driver conflict (purge nvidia packages).

### `hipblasConfig.cmake` not found during cmake
You are likely using ROCm 7.13 pre-release, which has incomplete packaging. Switch to the
stable ROCm 7.2.4 installer as described in Section 3.

### `cmath` / `cstdlib` not found during HIP compilation
GCC 12 dev headers are missing. Install:
```bash
sudo apt install libstdc++-12-dev g++-12
```
Then re-run cmake with the `--gcc-install-dir` flag as shown in Section 5.

### `rocminfo` shows no devices
Your user is not in the `render` and `video` groups. Run:
```bash
sudo usermod -aG render,video $USER
# then log out and back in (or reboot)
```
