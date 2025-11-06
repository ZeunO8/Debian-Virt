cmake_minimum_required(VERSION 3.16)
project(Debian-Virt VERSION 1.0 LANGUAGES NONE)

# Network options
set(ARM_NET_OPTS -net user,hostfwd=tcp:127.0.0.1:2223-:22 -net nic)
set(I386_NET_OPTS -net user,hostfwd=tcp:127.0.0.1:2224-:22 -net nic)

# Function to download ISO files if missing
function(download_iso_if_missing ISO_URL ISO_PATH)
  if(NOT EXISTS ${ISO_PATH})
    message(STATUS "Downloading ${ISO_URL} -> ${ISO_PATH}")
    file(DOWNLOAD ${ISO_URL} ${ISO_PATH} SHOW_PROGRESS)
  else()
    message(STATUS "Found existing ISO: ${ISO_PATH}")
  endif()
endfunction()

# Function for multi-arch QEMU setup
function(create_vm_targets ARCH ISO CPU MACHINE BOOTMODE)
  set(DISK_IMAGE "${CMAKE_BINARY_DIR}/debian13_${ARCH}.img")
  set(DISK_SIZE "10G")
  set(VM_RAM "3072")
  set(VM_CPUS "4")

  if("${ARCH}" STREQUAL "armhf")
    set(QEMU_BIN qemu-system-arm)
  else()
    set(QEMU_BIN qemu-system-${ARCH})
  endif()

  set(QEMU_IMG qemu-img)

  # Only create disk if missing (safe)
  add_custom_command(OUTPUT "${DISK_IMAGE}"
    COMMAND ${QEMU_IMG} create -f qcow2 "${DISK_IMAGE}" "${DISK_SIZE}"
    COMMENT "Creating ${ARCH} disk image (only if missing)"
  )
  add_custom_target(${ARCH}-img DEPENDS "${DISK_IMAGE}")

  if("${BOOTMODE}" STREQUAL "KERNEL")
    set(EXTRACT_DIR "${CMAKE_BINARY_DIR}/${ARCH}-iso")
    set(KERNEL_PATH "${EXTRACT_DIR}/install.ahf/vmlinuz")
    set(INITRD_PATH "${EXTRACT_DIR}/install.ahf/initrd.gz")

    add_custom_command(OUTPUT "${KERNEL_PATH}" "${INITRD_PATH}"
      COMMAND ${CMAKE_COMMAND} -E make_directory "${EXTRACT_DIR}"
      COMMAND 7z x -y "${ISO}" -o${EXTRACT_DIR}
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
      COMMENT "Extracting kernel+initrd from ${ISO} (requires 7z)"
      VERBATIM
    )
    add_custom_target(${ARCH}-extract DEPENDS "${KERNEL_PATH}" "${INITRD_PATH}")

    add_custom_target(${ARCH}-install
      COMMAND ${QEMU_BIN}
              -M ${MACHINE}
              -cpu ${CPU}
              -smp ${VM_CPUS}
              -m ${VM_RAM}
              -kernel "${KERNEL_PATH}"
              -initrd "${INITRD_PATH}"
              -append "console=ttyAMA0"
              -drive if=none,file="${DISK_IMAGE}",format=qcow2,id=hd0
              -device virtio-blk-device,drive=hd0
              -drive if=none,file="${ISO}",format=raw,readonly=on,id=cdrom0
              -device virtio-blk-device,drive=cdrom0
              -device virtio-net-device,netdev=net0
              -netdev user,id=net0,hostfwd=tcp:127.0.0.1:2223-:22
              -nographic
      DEPENDS "${DISK_IMAGE}" ${ARCH}-extract
      USES_TERMINAL
      COMMENT "Launching ${ARCH} installer (kernel+initrd+ISO+network)"
    )

    add_custom_target(${ARCH}-run
      COMMAND ${QEMU_BIN}
              -M ${MACHINE}
              -cpu ${CPU}
              -smp ${VM_CPUS}
              -m ${VM_RAM}
              -drive if=none,file="${DISK_IMAGE}",format=qcow2,id=hd0
              -device virtio-blk-device,drive=hd0
              -device virtio-net-device,netdev=net0
              -netdev user,id=net0,hostfwd=tcp:127.0.0.1:2223-:22
              -vga virtio
      DEPENDS "${DISK_IMAGE}"
      USES_TERMINAL
      COMMENT "Booting ${ARCH} VM (virtio disk + network)"
    )

  else()
    add_custom_target(${ARCH}-install
      COMMAND ${QEMU_BIN}
              -M ${MACHINE}
              -cpu ${CPU}
              -smp ${VM_CPUS}
              -m ${VM_RAM}
              -drive file="${DISK_IMAGE}",if=virtio,format=qcow2
              -cdrom "${ISO}"
              -boot d
	      -cpu host
	      -accel kvm
	      ${I386_NET_OPTS}
      DEPENDS "${DISK_IMAGE}"
      USES_TERMINAL
      COMMENT "Launching ${ARCH} installer from ISO"
    )

    add_custom_target(${ARCH}-run
      COMMAND ${QEMU_BIN}
              -M ${MACHINE}
              -cpu ${CPU}
              -smp ${VM_CPUS}
              -m ${VM_RAM}
              -drive file="${DISK_IMAGE}",if=virtio,format=qcow2
	      -cpu host
	      -accel kvm
	      ${I386_NET_OPTS}
      DEPENDS "${DISK_IMAGE}"
      USES_TERMINAL
      COMMENT "Booting ${ARCH} VM"
    )
  endif()
endfunction()

# ISO URLs
set(ARMHF_ISO_URL "https://cdimage.debian.org/debian-cd/current/armhf/iso-cd/debian-13.1.0-armhf-netinst.iso")
set(I386_ISO_URL  "https://cdimage.debian.org/mirror/cdimage/archive/12.1.0/i386/iso-cd/debian-12.1.0-i386-netinst.iso")
set(OBOS_ISO_URL  "http://obos-dev.ddns.net/images/obos-full.iso")

# ISO paths
set(ARMHF_ISO_PATH "${CMAKE_SOURCE_DIR}/debian-13.1.0-armhf-netinst.iso")
set(I386_ISO_PATH  "${CMAKE_SOURCE_DIR}/debian-12.1.0-i386-netinst.iso")
set(OBOS_ISO_PATH  "${CMAKE_SOURCE_DIR}/obos-full.iso")

# Download missing ISOs
download_iso_if_missing(${ARMHF_ISO_URL} ${ARMHF_ISO_PATH})
download_iso_if_missing(${I386_ISO_URL} ${I386_ISO_PATH})
download_iso_if_missing(${OBOS_ISO_URL} ${OBOS_ISO_PATH})

# Create VM targets
create_vm_targets(armhf ${ARMHF_ISO_PATH} "cortex-a15" "virt" "KERNEL")
create_vm_targets(i386  ${I386_ISO_PATH}  "pentium"    "pc"    "ISO")

# OBOS hobby OS target addition
add_custom_target(obos-run
  COMMAND qemu-system-x86_64
          -drive format=raw,file=${OBOS_ISO_PATH},media=disk,index=0
          -m 2048
          -M q35
          -smp 1
          -cpu host
          -accel kvm
	  -nic none
  USES_TERMINAL
  COMMENT "Booting OBOS"
)
add_custom_target(obos-debug
  COMMAND qemu-system-x86_64
          -drive format=raw,file=${OBOS_ISO_PATH},media=disk,index=0
          -m 2048
          -M q35
          -smp 1
          -cpu host
          -accel kvm
	  -nic none
	  -s -S
  USES_TERMINAL
  COMMENT "Booting OBOS (GDB Debug)"
)
