# Debian-Virt

A CMake-based script to install and run Debian virtual machines using QEMU.

## Prerequisites

- CMake (>= 3.16)
- QEMU (qemu-system-arm, qemu-system-i386, qemu-img)
- 7-Zip (for ARM kernel extraction)

## Usage

### 1. Configure the Project

```bash
cmake -B build
```

This will automatically download the required Debian ISO files if they don't exist.

### 2. Install Debian

Choose your architecture and run the installer:

**For ARM (armhf):**
```bash
cmake --build build --target armhf-install
```

**For i386:**
```bash
cmake --build build --target i386-install
```

This creates a 10GB disk image and launches the Debian installer in QEMU.

### 3. Run the Installed System

After installation is complete, boot the installed system:

**For ARM (armhf):**
```bash
cmake --build build --target armhf-run
```

**For i386:**
```bash
cmake --build build --target i386-run
```

## Network Access

SSH is available on localhost port 2223:
```bash
ssh -p 2223 user@127.0.0.1
```

## VM Configuration

- **Disk Size:** 10GB (qcow2 format)
- **RAM:** 4096MB
- **CPUs:** 4 cores
- **Network:** User-mode networking with SSH port forwarding (2223 → 22)
