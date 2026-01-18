Complete Guide: AMD RX 570 + Dell OptiPlex 7040 ROCm Setup
1. Hardware & BIOS Prerequisites
Before modifying software, ensure the Dell motherboard is ready.

Power: Ensure you are using an external PSU or a SATA-to-8pin adapter (if the card is low-power) that can handle the 120W+ draw of the RX 570.

BIOS Settings: * Set Secure Boot to Disabled (to allow the amdgpu-dkms driver to load).

Set Intel Virtualization Technology for Direct I/O (VT-d) to Enabled.

2. Kernel Fix: Enable PCIe Atomics
The RX 570 requires PCIe Atomic operations. Intel chipsets in the OptiPlex 7040 block these by default. Enabling "Pass-Through" mode is mandatory.

Open the GRUB file:

Bash
sudo nano /etc/default/grub
Locate the line starting with GRUB_CMDLINE_LINUX_DEFAULT.

Modify it to include iommu=pt:

Plaintext
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=pt"
Update GRUB and reboot:

Bash
sudo update-grub
sudo reboot
3. Install ROCm 5.7.3 (Ubuntu 22.04)
ROCm 5.7.3 is the most stable version for the Polaris (gfx803) architecture.

Bash
# 1. System Update
sudo apt update && sudo apt upgrade -y
sudo apt install wget gnupg2 build-essential dkms linux-headers-$(uname -r) -y

# 2. Add AMDGPU Repository Key
sudo mkdir --parents /var/lib/amdgpu-install
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null

# 3. Add ROCm 5.7.3 Repository
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/5.7.3 jammy main" \
    | sudo tee /etc/apt/sources.list.d/rocm.list

# 4. Install Driver and OpenCL Stack
sudo apt update
sudo apt install amdgpu-dkms rocm-opencl-sdk rocm-opencl-runtime -y

# 5. Set Permissions
sudo usermod -aG video $USER
sudo usermod -aG render $USER
(Reboot again if you haven't recently to apply group permissions).

4. Environment Overrides (The "Unlocking" Step)
The driver knows the GPU is there but hides it because it is "Legacy." You must force it to appear.

Create a permanent environment file in your project folder: File: ~/my_opencl/.env

Plaintext
# Force ROCm to treat RX 570 (gfx803) as supported
HSA_OVERRIDE_GFX_VERSION=8.0.3
ROC_ENABLE_PRE_VEGA=1

# Bypass hardware platform checks
HSA_IGNORE_AMD_PLATFORM_CHECK=1

# Force Python to use the ROCm library path
LD_LIBRARY_PATH=/opt/rocm/lib
5. Python Project Setup
Inside your VS Code terminal, set up the virtual environment and install the OpenCL bridge.

Bash
cd ~/my_opencl
python3 -m venv .venv
source .venv/bin/activate
pip install pyopencl ocl-icd-system
6. Verification Script (test_gpu.py)
Run this to confirm the device is visible and initialized.

Python
import pyopencl as cl
import os

# Ensure overrides are active
os.environ['HSA_OVERRIDE_GFX_VERSION'] = '8.0.3'
os.environ['ROC_ENABLE_PRE_VEGA'] = '1'

try:
    platforms = cl.get_platforms()
    gpu = platforms[0].get_devices()[0]
    print(f"✅ Success! GPU Found: {gpu.name}")
    print(f"Compute Units: {gpu.max_compute_units}")
except Exception as e:
    print(f"❌ Failed: {e}")
7. Performance Troubleshooting
If you do not hit the 4.35 TFLOPS mark:

Check Power: If the GPU isn't getting enough watts, it will downclock.

Check Thermal Throttling: Use watch -n 1 rocm-smi to monitor temperatures.

Check CPU Bottleneck: The OptiPlex 7040's i5/i7 6th gen is plenty for OpenCL, but ensure "High Performance" power mode is active in Ubuntu settings.
