Building OPNsense 24.1 virtual machine disk images
==================================================

This fork of [opnsense/tools](https://github.com/opnsense/tools) adds a brief tutorial and two devices, `AMD64VM` and `ARM64VM`.
These devices allow building [OPNsense](https://opnsense.org/) VM images (amd64 and aarch64) with the console preset to EFI or serial instead of the default VGA console (System: Settings: Administration: Console).

This tutorial explains how to set up a virtualized build system from scratch and build a disk image.
Details about all build steps and options can be found in the official [opnsense/tools README](https://github.com/opnsense/tools/blob/master/README.md).

Unofficial sample VM images are published in the [GitHub releases](https://github.com/maurice-w/opnsense-vm-images/releases) section of this repository.

Setting up a build system
=========================

Download a [FreeBSD-13.2-RELEASE VM image](https://download.freebsd.org/releases/VM-IMAGES/13.2-RELEASE/) matching the CPU architecture (amd64 or aarch64) and hypervisor.
Extract the disk image and expand its size to at least 40 GB (Hyper-V: `Resize-VHD`, QEMU: `qemu-img resize`).
Attach the image to a VM, boot it and log in as root using the console (SSH is disabled by default).

Set a root password, enable SSH and update FreeBSD:

    # passwd
    # echo 'sshd_enable="YES"' >> /etc/rc.conf && echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
    # setenv PAGER cat && freebsd-update fetch install && shutdown -r now

After the reboot, the remaining steps can be performed via SSH or the local console.

Install Git and clone this repository:

    # pkg install -y git && git clone https://github.com/maurice-w/opnsense-vm-images /usr/tools && cd /usr/tools

Building a VM image
===================

The first argument of the `vm` command specifies the disk image format. All formats supported by [mkimg(1)](https://man.freebsd.org/cgi/man.cgi?query=mkimg) are available.

All VM images have a GUID partition table (GPT) and an EFI system partition (ESP) and support UEFI boot. Images for amd64 additionally support legacy BIOS boot.

Root and swap partition sizes can be customized with the second and third argument of the `vm` command.
The minimum root partition size is 3 GB. The swap partition can be omitted by setting the third argument to `never`.

The fourth argument of the `vm` command specifies the default console (EFI or serial). For a VGA console, this parameter must be omitted.

By default, the root partition uses the UFS file system. For a ZFS file system, the `ZFS=zpool` build option must be added.

Building from source
--------------------

All [tagged OPNsense versions](https://github.com/opnsense/core/tags) can be build by setting the `VERSION` option accordingly.

VHDX image (Hyper-V), 3 GB root partition, no swap partition, EFI console, OPNsense 24.1.r1-amd64, UFS file system:

    # make update vm-vhdx,3G,never,efi SETTINGS=24.1 VERSION=24.1.r1 DEVICE=AMD64VM

QCOW2 image (QEMU), 30 GB root partition, 2 GB swap partition, serial console, OPNsense 24.1-amd64, ZFS file system:

    # make update vm-qcow2,30G,2G,serial SETTINGS=24.1 VERSION=24.1 DEVICE=AMD64VM ZFS=zpool

QCOW2 image (QEMU), 16 GB root partition, no swap partition, EFI console, OPNsense 24.1.1-aarch64, UFS file system:

    # make update vm-qcow2,16G,never,efi SETTINGS=24.1 VERSION=24.1.1 DEVICE=ARM64VM

QCOW2 image (QEMU), 20 GB root partition, 1 GB swap partition, serial console, OPNsense 24.1.2-aarch64, UFS file system:

    # make update vm-qcow2,20G,1G,serial SETTINGS=24.1 VERSION=24.1.2 DEVICE=ARM64VM

Using prefetched sets
---------------------

This method is much faster, but requires pre-compiled base, kernel and packages sets. The `VERSION` option specifies which version of the sets to download.
[Official packages sets](https://pkg.opnsense.org/FreeBSD:13:amd64/24.1/sets/) are only published for some releases.
Since the official mirrors only offer amd64 sets, a custom mirror needs to be specified for prefetching aarch64 sets.

VHDX image (Hyper-V), 8 GB root partition, no swap partition, EFI console, OPNsense 24.1.r1-amd64, ZFS file system:

    # make update prefetch-base,kernel,packages vm-vhdx,8G,never,efi SETTINGS=24.1 VERSION=24.1.r1 DEVICE=AMD64VM ZFS=zpool

QCOW2 image (QEMU), 40 GB root partition, 4 GB swap partition, VGA console, OPNsense 24.1-amd64, UFS file system:

    # make update prefetch-base,kernel,packages vm-qcow2,40G,4G SETTINGS=24.1 VERSION=24.1 DEVICE=AMD64VM

QCOW2 image (QEMU), 10 GB root partition, no swap partition, EFI console, OPNsense 24.1.r1-aarch64, UFS file system:

    # make update prefetch-base,kernel,packages vm-qcow2,10G,never,efi SETTINGS=24.1 VERSION=24.1.r1 DEVICE=ARM64VM MIRRORS=https://opnsense-update.walker.earth

QCOW2 image (QEMU), 5 GB root partition, no swap partition, serial console, OPNsense 23.7-aarch64, UFS file system:

    # make update prefetch-base,kernel,packages vm-qcow2,5G,never,serial SETTINGS=24.1 VERSION=24.1 DEVICE=ARM64VM MIRRORS=https://opnsense-update.walker.earth

Downloading the VM image
========================

If successful, the VM image can be found under:

    # make print-IMAGESDIR

Download the image file:

    # scp root@[2001:db8::a]:/usr/local/opnsense/build/24.1/[amd64|aarch64]/images/OPNsense-24.1[.x]-vm-[amd64|aarch64].[qcow2|vhdx|...] .

Caveats
=======

- There are no official FreeBSD VHDX images. If setting up a build system in a Hyper-V Generation 2 VM, the VHD image must be converted to VHDX first (`Convert-VHD`).
- FreeBSD doesn't support Secure Boot, this needs to be disabled in the VM settings.
- The EFI / VGA console of the FreeBSD VM images defaults to a US keyboard layout. If required, this can be temporarily changed:

      # kbdmap

- Out of the box, FreeBSD only supports SLAAC, not DHCPv6.
  [dhcpcd(8)](https://man.freebsd.org/cgi/man.cgi?query=dhcpcd) can be used if SLAAC isn't available:

      # pkg install -y dhcpcd && echo 'dhclient_program="/usr/local/sbin/dhcpcd"' >> /etc/rc.conf

  In IPv6-only networks without SLAAC, a temporary address needs to be added first to allow installing dhcpcd(8):

      # ifconfig [hn0|vtnet0] inet6 2001:db8::a

- Building OPNsense VHDX images is supported (`vm-vhdx`), but the VHDX file is as large as the specified partition sizes combined.
  Setting the root partition size to 3 GB and expanding the finished VHDX image to the desired size (`Resize-VHD`) is recommended.
