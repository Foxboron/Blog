---
title: "mkinitcpio and UEFI stubs"
date: "2021-08-22T00:00:00+02:00"
draft: true
tags:
- english
- archlinux
- mkinitcpio
---

# mkinitcpio v31 and UEFI stubs

A few months ago I wrote up some code for `mkinitcpio` which teaches it how to
create UEFI executables utilizing the systemd stub.

The change can be found here: https://github.com/archlinux/mkinitcpio/pull/53

This is a short introduction to why the feature is great, how it makes it easier
to boot your system, and how it can be used to better secure your system with
something like secure boot.

### The Boot Process

For the past decade most computers have two ways to boot. The legacy BIOS mode
and UEFI which is suppose to replace it. It frankly does a lot of things, but
one of the more interesting aspects is that the Linux kernel is a valid [MS DOS
binary](https://en.wikipedia.org/wiki/DOS_MZ_executable). If you read out the
two first bytes you will see `MZ`.

The reason for this is that when we launch Linux from UEFI we are actually
running the Linux binary with a bunch of commands which makes out our entry
point. Because UEFI is itself a boot loader you can use this fact to boot Linux
directly from UEFI as an [UEFI boot entry](https://wiki.archlinux.org/title/EFISTUB#efibootmgr).

However, most of us don't want to mess with UEFI directly so we use a bootloader
like `grub` or `systemd-boot` because it's easier to deal with.

### Securing the boot chain

When we set a bootloader we usually provide a configuration file, the
initramfs, and the kernel binary. The initramfs is essentially a "stage 0"
linux distribution responsible for unlocking encrypted partitions, mounting
your filesystem and other partitions and then launch the init binary. All these 3
files lie unencrypted on your boot partition[[^1]]. With Secure Boot we could
sign the kernel, as it is an UEFI executable. However, this leaves our boot
configuration and initramfs completely unprotected.

The solution to this is using a binary that lets us embed all these parts into
one binary. [EFI unified kernel images](https://systemd.io/BOOT_LOADER_SPECIFICATION/#type-2-efi-unified-kernel-images)
essentially allows you to accomplish this in an (almost) straight forward
way.


### UEFI Stubs

`systemd` provides the stub binary on most distributions, if you don't have
`systemd` packaged you might have it as part of the `gummiboot` package.

The way it works is that we are inserting the data we need into sections in the
binary file, which is then picked up by the stub executable.

{{< highlight shell >}}
#!/bin/bash
objcopy \
    --add-section .osrel="/etc/os-release" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="/etc/kernel/cmdline" --change-section-vma .cmdline=0x30000 \
    --add-section .splash="/usr/share/systemd/bootctl/splash-arch.bmp" --change-section-vma .splash=0x40000 \
    --add-section .linux="/boot/vmlinuz-linux" --change-section-vma .linux=0x2000000 \
    --add-section .initrd=<(cat /boot/intel-ucode.img /boot/initrd-linux.img) --change-section-vma .initrd=0x3000000 \
    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/efi/EFI/Linux/linux.efi"
{{< /highlight >}}

This Arch Linux specific example would create a binary which has the
distribution information (the `os-release` file), the kernel cmdline read from a
file, a cool bmp file with your distributions logo, the kernel, and the initramfs
with the microcode bundled.

Signing this file would then help you authenticate most of the files which is
used as part of your boot process. This file could then be executed from
your UEFI shell without any additional command line arguments, or directly used by your bootloader[[^2]].

All of this is fairly simple, but because of the security implications a lot of
tooling implement this on their own in different variations. Having a unified
way of generating these files helps a lot in this case.

### mkinitcpio

`mkinitcpio` is the initramfs generator mainly used and developed by Arch Linux.
Some parts of this section are thus a bit distribution specific. However similar
features exist in `dracut` with `--uefi` as an example. If your local initramfs
generator doesn't support this feature it's a fairly straight forward feature
you could contribute to the project!

If you want to follow along with the example below you can fetch the release
candidate from the project host. Any usability, documentation or code changes
are more than welcome!

https://github.com/archlinux/mkinitcpio/releases/tag/v31_rc0

First off we are going to change the `linux.preset` file which denotes the
configuration for the Linux kernel on Arch.

{{< highlight diff >}}
--- /etc/mkinitcpio.d/linux.preset
+++ /etc/mkinitcpio.d/linux.preset
@@ -2,13 +2,16 @@

 ALL_config="/etc/mkinitcpio.conf"
 ALL_kver="/boot/vmlinuz-linux"
+ALL_microcode=(/boot/*-ucode.img)

 PRESETS=('default' 'fallback')

 #default_config="/etc/mkinitcpio.conf"
 default_image="/boot/initramfs-linux.img"
-#default_options=""
+default_efi_image="/efi/EFI/Linux/archlinux-linux.efi"
+default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

 #fallback_config="/etc/mkinitcpio.conf"
 fallback_image="/boot/initramfs-linux-fallback.img"
-fallback_options="-S autodetect"
+fallback_efi_image="/efi/EFI/Linux/archlinux-linux-fallback.efi"
+fallback_options="-S autodetect --splash /usr/share/systemd/bootctl/splash-arch.bmp"
{{< /highlight >}}

This just tells `mkinitcpio` where to find the microcode, and the filename we
want for the executable. We are also passing `--splash` as an option for the
boot splash image. Note that you need to specify the save location to where
your current EFI boot partition currently is mounted.

Next up is fixing the kernel cmdline. By default `mkinitcpio` is going to be
reading from `/etc/kernel/cmdline`.  If you are unsure what your current kernel
cmdline is you can inspect `/proc/cmdline` and use it as a starting point.
However, be mindful that `initrd` entires pointing at microcode and the
initramfs needs to be removed.

{{< highlight text >}}
# cat /etc/kernel/cmdline
rw quiet bgrt_disable
{{< /highlight >}}

The file should be similar to the above. Also do note that any `root=` or
`cyptdevice=` flags are still needed if you are not running a systemd enabled
initramfs with [discoverable partitions](https://systemd.io/DISCOVERABLE_PARTITIONS/).

We are also adding `bgrt_disable` to the kernel cmdline. This is a [recent
flag](https://lore.kernel.org/linux-acpi/20200304225529.6706-1-alex.hung@canonical.com/T/)
which tells Linux to not display the OEM logo after loading the ACPI tables. It
will make the splash image show for a few more seconds instead of being
overwritten by the some ugly logo during boot.

When running `mkinitcpio -P` you should see something similar to the output
below.

{{< highlight text >}}
[..snip..]
==> Starting build: 5.13.10-arch1-1
  -> Running build hook: [base]
  -> Running build hook: [systemd]
  -> Running build hook: [autodetect]
  -> Running build hook: [modconf]
  -> Running build hook: [block]
  -> Running build hook: [keyboard]
  -> Running build hook: [sd-encrypt]
  -> Running build hook: [filesystems]
==> Generating module dependencies
==> Creating zstd-compressed initcpio image: /boot/initramfs-linux.img
==> Image generation successful
==> Creating UEFI executable: /efi/EFI/Linux/archlinux-linux.efi
  -> Using UEFI stub: /usr/lib/systemd/boot/efi/linuxx64.efi.stub
  -> Using kernel image: /lib/modules/5.13.10-arch1-1/vmlinuz
  -> Using os-release file /etc/os-release
==> UEFI executable generation successful
{{< /highlight >}}

Tada! We have an UEFI stub generated from `mkinitcpio`!

If you are using `systemd-boot` you don't need to configure anything else. The
bootloader is going to be looking into the `EFI/Linux` directory for valid
bootable UEFI stubs to display in the menu. This make setting up the bootloader
a lot simpler as we only need to run `bootctl install` and generate the binary
to have a working bootloader.


## Tips and Tricks

If you want to keep older around kernels this feature also makes it extremely
simple. Extract the package version of the `linux` package when creating the
image. If you use systemd-boot this is going to be bootable without any further
configuration.

{{< highlight text >}}
default_efi_image="/efi/EFI/Linux/linux-$(pacman -Q linux | awk '{print $2}').efi"
{{< /highlight >}}

[^1]: Yes, some people do encrypt their boot partitions.

[^2]: I feel like it's worth mentioning that some bootloaders do not actually
validate secure boot signatures on the files they run. `systemd-boot` does,
and `grub` doesn't. Make of that what you will.
