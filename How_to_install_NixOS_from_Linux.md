---
title: How to install NixOS from Linux
permalink: /How_to_install_NixOS_from_Linux/
---

How to install NixOS from Linux
-------------------------------

You already have a running linux and a functional grub on your primary partition and you don't want to waste a CD-R as you feel that you really don't need to? Right. We also assume that you have a "spare" partition where to install NixOS ready.

to simplify:

/dev/sda1 : your boot partition, containing already working grub

/dev/sda2 : your linux root partition, containing your currently working OS

/dev/sda3 : your spare partition to where you will install NixOS

the_iso : the nixos livecd iso

~/some_dir : directory where "the_iso" is (loop) mounted

/boot : the boot directory, where grub is installed (/dev/sda1 and /dev/sda2 \*can\* be the same partition!)

### Unpacking the ISO image

-   obtain the ISO "the_iso"
-   mount -o loop "the_iso" "~/some_dir"
-   (mount the /boot partition, containing your working grub)
-   cp some_dir/boot/bzImage /boot/nixos-livecd-bzImage
-   cp some_dir/boot/initrd /boot/nixos-livecd-initrd
-   cp some_dir/nix-store.squashfs /nix-store.squashfs

### Modifying your bootloader's config

Look at some_dir/boot/grub/grub.cfg. This is Grub-2 main config file of the ISO. Locate the NixOS menuentry section:

    menuentry "NixOS Installer / Rescue" {
      linux /boot/bzImage init=/nix/store/p5n72ay1c1wx4wry90zabr8jnljpdzgx-nixos-0.2pre4601_1def5ba-48a4e91/init root=LABEL=NIXOS_0.2pre4601_1def5ba-48a4e91
      initrd /boot/initrd
    }

The goal is to tell your bootloader to boot /nixos-livecd-bzImage with correct init argument.

#### Grub &lt; 2

To setup grub-1, edit your /boot/grub/menu.lst (or equivalent). Add following lines to the config:

    title NixOS LiveCD
    kernel /nixos-livecd-bzImage init=/nix/store/p5n72ay1c1wx4wry90zabr8jnljpdzgx-nixos-0.2pre4601_1def5ba-48a4e91/init root=/dev/sda2 splash=verbose vga=0x317
    initrd /nixos-livecd-initrd

Note, that hash should match with what you have seen in some_dir/grub/grub.cfg

Go to the reboot section

#### Other bootloaders

Should also work. Please add instructions here.

### Booting into LiveCD

Reboot. Select "NixOS LiveCD" from the bootloader menu. If everything is OK, you will see Login prompt asking you to login as root with empty password. ''DO NOT TRUST IT BLINDLY. You probably have your /etc mounted from /dev/sda2 so it contains your old passwd (as well as LiveCD stuff merged into by the means of UnionFS). So if empty password is not working, try your old root password.

Thats it. Now follow the manual (Alt-F8), mount /dev/sda3 as /mnt, do nixos-config and so on.