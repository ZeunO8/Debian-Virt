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

### 4. Running OBOS

The project also includes targets for running OBOS, an operating system developed by [Oberrow](https://github.com/OBOS-dev/obos):

**Run OBOS:**
```bash
cmake --build build --target obos-run
```

**Run OBOS with GDB debugging:**
```bash
cmake --build build --target obos-debug
```

The OBOS ISO will be automatically downloaded during project configuration.

## Network Access

SSH is available on localhost for Debian VMs:
- **ARM (armhf):** Port 2223
- **i386:** Port 2224

```bash
ssh -p 2223 user@127.0.0.1  # For ARM
ssh -p 2224 user@127.0.0.1  # For i386
```

## VM Configuration

### Debian VMs
- **Disk Size:** 10GB (qcow2 format)
- **RAM:** 3072MB
- **CPUs:** 4 cores
- **Network:** User-mode networking with SSH port forwarding

### OBOS VM
- **RAM:** 2048MB
- **CPUs:** 1 core
- **Acceleration:** KVM enabled
- **Network:** Disabled
