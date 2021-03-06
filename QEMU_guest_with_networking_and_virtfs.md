---
title: QEMU guest with networking and virtfs
permalink: /QEMU_guest_with_networking_and_virtfs/
---

Introduction
------------

This article describes how to run a QEMU guest with full networking and VirtFS support. VirtFS (Plan 9 folder sharing over Virtio - I/O virtualization framework) allows you to mount folders from the host system on the guest system, this is significantly faster than folder sharing over the network e.g. NFS.

Enable kernel modules
---------------------

The NixOS install process will typically enable the relevant kvm modules for your system (kvm-amd or kvm-intel) in hardware-configuration.nix (). You can also add "tun" and "virtio" if you wish to use the networking setup described below.

`boot.kernelModules = [ "kvm-amd" "tun" "virtio" ];`

You can check for the existence of /dev/kvm after rebooting. You may also need to enable hardware virtualization in your BIOS or similar.

Prepare a guest image
---------------------

`qemu-img create -f qcow2 /mnt/OSImages/mint.qcow2 5G`
`qemu-kvm -m 1024 -drive file=/mnt/OSImages/mint.qcow2  -cdrom /mnt/OSImages/mint-livecd.iso -boot d`

The default amount of memory available to the guest is 128MB so you will probably want to specify something more than that with the "m" flag. By default the guest will have access to the network so you can install the OS at this stage if you like.

Alternatively you can download a QEMU image with an OS already installed e.g. Debian: <http://people.debian.org/~aurel32/qemu/>

`qemu-kvm -m 1024 -drive file=/home/goibhniu/Downloads/debian_squeeze_amd64_standard.qcow2`

Prepare host network
--------------------

There are a number of options to enable access to the guest OS from the host, using [VDE](http://vde.sourceforge.net/) (Virtual Distributed Ethernet) is probably the easiest and most versatile.

As root add a "kvm" group so that you only have to do the networking setup as root:

`$ groupadd kvm`
`$ usermod -G kvm username`

Start the vde daemon

`$ vde_switch -tap tap0 -mod 660 -group kvm -daemon`
`$ ip addr add 10.0.2.1/24 dev tap0`
`$ ip link set dev tap0 up`

You should now see the tap0 interface with \`ip addr\`/\`ifconfig\` and it should have been assigned the IP address specified. It is important that you don't use the same subnet as your host network interface (eth0/wlan0) is using.

Forward traffic:

`$ sysctl -w net.ipv4.ip_forward=1`
`$ iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -j MASQUERADE`

References:

-   [Building a Virtual Army](http://blog.falconindy.com/articles/build-a-virtual-army.html)
