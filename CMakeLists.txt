cmake_minimum_required(VERSION 3.16)
project(Debian-Virt VERSION 1.0 LANGUAGES NONE)

if(WIN32)
  set(NET_OPTS -net user,hostfwd=tcp:127.0.0.1:2223-:22 -net nic)
else()
  set(NET_OPTS -net user,hostfwd=tcp:127.0.0.1:2223-:22 -net nic)
endif()

function(create_vm_targets ARCH ISO CPU MACHINE BOOTMODE)
  set(DISK_IMAGE "${CMAKE_BINARY_DIR}/debian13_${ARCH}.img")
  set(DISK_SIZE "10G")
  set(VM_RAM "4096")
  set(VM_CPUS "4")
  if("${ARCH}" STREQUAL "armhf")
    set(QEMU_BIN qemu-system-arm)
  else()
    set(QEMU_BIN qemu-system-${ARCH})
  endif()
  set(QEMU_IMG qemu-img)

  add_custom_target(${ARCH}-img
    COMMAND ${QEMU_IMG} create -f qcow2 "${DISK_IMAGE}" "${DISK_SIZE}"
    BYPRODUCTS "${DISK_IMAGE}"
    COMMENT "Creating ${ARCH} disk image"
  )

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
              -drive file="${DISK_IMAGE}",if=virtio,format=qcow2
              -cdrom "${ISO}"
              ${NET_OPTS}
              -nographic
      DEPENDS ${ARCH}-img ${ARCH}-extract
      USES_TERMINAL
      COMMENT "Launching ${ARCH} installer (kernel+initrd)"
    )

    add_custom_target(${ARCH}-run
      COMMAND ${QEMU_BIN}
              -M ${MACHINE}
              -cpu ${CPU}
              -smp ${VM_CPUS}
              -m ${VM_RAM}
              -drive file="${DISK_IMAGE}",if=virtio,format=qcow2
              ${NET_OPTS}
              -nographic
      DEPENDS ${ARCH}-img
      USES_TERMINAL
      COMMENT "Booting ${ARCH} VM"
    )

  else()  # BOOTMODE == ISO (BIOS/UEFI-able)
    add_custom_target(${ARCH}-install
      COMMAND ${QEMU_BIN}
              -M ${MACHINE}
              -cpu ${CPU}
              -smp ${VM_CPUS}
              -m ${VM_RAM}
              -drive file="${DISK_IMAGE}",if=virtio,format=qcow2
              -cdrom "${ISO}"
              -boot d
              ${NET_OPTS}
      DEPENDS ${ARCH}-img
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
              ${NET_OPTS}
      DEPENDS ${ARCH}-img
      USES_TERMINAL
      COMMENT "Booting ${ARCH} VM"
    )
  endif()
endfunction()

create_vm_targets(armhf "${CMAKE_SOURCE_DIR}/debian-13.1.0-armhf-netinst.iso" "cortex-a15" "virt" "KERNEL")
create_vm_targets(i386  "${CMAKE_SOURCE_DIR}/debian-12.1.0-i386-netinst.iso"   "pentium"    "pc"    "ISO")