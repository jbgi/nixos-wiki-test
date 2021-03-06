---
title: Luks-based FDE with Yubikey PBA and btrfs on UEFI NixOS
permalink: /Luks-based_FDE_with_Yubikey_PBA_and_btrfs_on_UEFI_NixOS/
---

This page is a minimalistic guide for setting up luks-based full disk encryption with YubiKey pre-boot authentication on a UEFI system. The YubiKey PBA in NixOS currently features two-factor authentication using a (secret) user passphrase and a YubiKey in challenge-response mode.

Requirements
------------

-   A NixOS live system (at least a recent 14.02pre) booted in UEFI mode on the target machine.
-   A YubiKey Standard plugged into the target machine with a free configuration slot (that will be overwritten).

Partitioning
------------

Create a GPT partition table and two partitions on the target disk.

-   Partition 1: This will be the EFI system partition: 100MB-300MB
-   Partition 2: This will be the Luks-encrypted partition, aka the "luks device": Rest of your disk

In the following we will use variables for identification, so set them to match your setup, e.g. like this:

    EFI_PART=/dev/sda1
    LUKS_PART=/dev/sda2

Setup the luks device
---------------------

**Step 1**: Create the necessary filesystem on the efi system partition, which will store the current salt for the PBA, and mount it.

    EFI_MNT=/root/boot
    mkdir "$EFI_MNT"
    mkfs.vfat -F 32 -n uefi "$EFI_PART"
    mount "$EFI_PART" "$EFI_MNT"

**Step 2**: Decide where on the efi system partition to store the salt and prepare the directory layout accordingly.

    STORAGE=/crypt-storage/default
    mkdir -p "$(dirname $EFI_MNT$STORAGE)"

**Step 3**: Install the packages required by the next steps to the live system and make two bash helper functions available.

Packages:

-   A C compiler, e.g. gcc
-   The YubiKey Personalization command line tool
-   OpenSSL

<!-- -->

    nix-env -i gcc-wrapper
    nix-env -i ykpers
    nix-env -i openssl

Helper functions:

-   Convert a raw binary string to a hexadecimal string
-   Convert a hexadecimal string to a raw binary string

<!-- -->

    rbtohex() {
        ( od -An -vtx1 | tr -d ' \n' )
    }

    hextorb() {
        ( tr '[:lower:]' '[:upper:]' | sed -e 's/\([0-9A-F]\{2\}\)/\\\\\\x\1/gI'| xargs printf )
    }

**Step 4**: Compile a small C program (shipped with NixOS) that allows for direct access to OpenSSL's PBKDF2 implementation.

    cc -O3 -I$(find / | grep "openssl/evp\.h" | head -1 | sed -e 's|/openssl/evp\.h$||g' | tr -d '\n') \
      -L$(find / | grep "lib/libcrypto" | head -1 | sed -e 's|/libcrypto\..*$||g' | tr -d '\n') \
      $(find / | grep "pbkdf2-sha512\.c" | head -1 | tr -d '\n') -o ./pbkdf2-sha512 -lcrypto

**Step 5**: Gather the initial salt for the PBA (set its length to what you find time-feasible on your machine).

    SALT_LENGTH=16
    salt="$(dd if=/dev/random bs=1 count=$SALT_LENGTH 2>/dev/null | rbtohex)"

**Step 6**: Gather the random secret key for the YubiKey (Important: This key should ***not*** be stored anywhere other than the YubiKey, as that poses a security risk).

    k_yubi="$(dd if=/dev/random bs=1 count=20 2>/dev/null | rbtohex)"

**Step 7**: Get the user passphrase used as the second factor in the PBA.

    read -s k_user

**Step 7**.5: Make ***very*** sure, that $k_user contains the correct user passphrase, or you will ***not*** be able to access your system after shutting down the live system.

**Step 8**: Calculate the initial challenge to the YubiKey.

    challenge="$(echo -n $salt | openssl dgst -binary -sha512 | rbtohex)"

**Step 9**: Calculate the response the YubiKey should give to that challenge (Only possible because right now we still know the secret key for the YubiKey).

    response="$(echo -n $challenge | hextorb | openssl dgst -binary -sha1 -mac HMAC -macopt hexkey:$k_yubi | rbtohex)"

**Step 10**: Derive the Luks slot key from the two factors.

-   Set the length of the Luks slot key and the ciphter appropriately.

` As an example, we will use AES-256, so we set the Luks device slot key length to 512 bit.`

-   Set the iteration count used for PBKDF2 to a high value still time-feasible for your machine.

<!-- -->

    KEY_LENGTH=512
    ITERATIONS=1000000
    k_luks="$(echo -n $k_user | ./pbkdf2-sha512 $(($KEY_LENGTH / 8)) $ITERATIONS $response | rbtohex)"

If you choose to authenticate without a user passphrase (not recommended), use this instead of the line above

    k_luks="$(echo | ./pbkdf2-sha512 $(($KEY_LENGTH / 8)) $ITERATIONS $response | rbtohex)"

**Step 11**: Create the luks device.

-   Set the cipher used by Luks appropriately
-   Set the has used by Luks appropriately

<!-- -->

    CIPHER=aes-xts-plain64
    HASH=sha512
    echo -n "$k_luks" | hextorb | cryptsetup luksFormat --cipher="$CIPHER" \
      --key-size="$KEY_LENGTH" --hash="$HASH" --key-file=- "$LUKS_PART"

**Step 12**: Store the salt and iteration count to the efi systems partition.

    echo -ne "$salt\n$ITERATIONS" > $EFI_MNT$STORAGE

**Step 13**: Store the secret key for the YubiKey on the YubiKey and configure it accordingly.

-   Set the slot used on the YubiKey appropriately

<!-- -->

    SLOT=2
    ykpersonalize -"$SLOT" -ochal-resp -ochal-hmac -a"$k_yubi"

**Step 14**: Open the luks device with the initial slot key.

-   Set the name for the luks device appropriately

<!-- -->

    LUKSROOT=luksroot
    echo -n "$k_luks" | hextorb | cryptsetup luksOpen --key-file=- "$LUKS_PART" "$LUKSROOT"

**Step 15**: Unmount the efi system partition.

    umount "$EFI_MNT"

LVM setup
---------

**Step 1**: Setup the Luks device as a physical volume.

    pvcreate "/dev/mapper/$LUKSROOT"

**Step 2**: Setup a volume group on the Luks device.

-   Set the name for the volume group appropriately

<!-- -->

    VGNAME=partitions
    vgcreate "$VGNAME" "/dev/mapper/$LUKSROOT"

**Step 3**: Setup two logical volumes on the Luks device.

-   Volume 1: This will be the swap partition: choose appropriate size, 2GB for example
-   Volume 2: This will be the main btrfs volume, of which all filesystem partitions will be subvolumes: Rest of the free space

<!-- -->

    lvcreate -L 2G -n swap "$VGNAME"
    FSROOT=fsroot
    lvcreate -l 100%FREE -n "$FSROOT" "$VGNAME"

    vgchange -ay

**Step 4**: Create the swap filesystem.

    mkswap -L swap /dev/partitions/swap

Btrfs setup
-----------

**Step 1**: Create the main btrfs volume's filesystem.

    mkfs.btrfs -L "$FSROOT" "/dev/partitions/$FSROOT"

Should the above fail, you might have encountered a bug that can be solved with doing the following, then attempting the above again:

    mkdir /mnt-root
    touch /mnt-root/nix-store.squashfs

**Step 2**: Mount the main btrfs volume.

    mount "/dev/partitions/$FSROOT" /mnt

**Step 3**: Create the subvolumes, for example "root" and "home".

    cd /mnt
    btrfs subvolume create root
    btrfs subvolume create home

**Step 4**: Create mountpoints on the root subvolume and finalise things for NixOS installation.

    umount /mnt
    mount -o subvol=root "/dev/partitions/$FSROOT" /mnt

    mkdir /mnt/home
    mount -o subvol=home "/dev/partitions/$FSROOT" /mnt/home

    mkdir /mnt/boot
    mount "EFI_PART" /mnt/boot

    swapon /dev/partitions/swap

NixOS installation
------------------

Configure and install NixOS as you normally would, with these changes:

-   Remove all detected filesystems from hardware-configuration.nix.
-   Add the following to your configuration.nix (Replace anything that looks like a Bash variable with the value that it currently holds for in your shell and modify as needed):

<!-- -->

      # Minimal list of modules to use the efi system partition and the YubiKey
      boot.initrd.kernelModules = [ "vfat" "nls_cp437" "nls_iso8859-1" "usbhid" ];

      # Crypto setup, set modules accordingly
      boot.initrd.luks.cryptoModules = [ "aes" "xts" "sha512" ];

      # Enable support for the YubiKey PBA
      boot.initrd.luks.yubikeySupport = true;
      # Configuration to use your Luks device
      boot.initrd.luks.devices = [ {
        name = "luksroot";
        device = "LUKS_PART";
        preLVM = true;
        yubikey = {
          storage = {
            device = $EFI_PART;
          };
        };
      } ];

      # File systems
      swapDevices = [ { device = "/dev/partitions/swap"; } ];

      fileSystems."/" = {
        label = "root";
        device = "/dev/partitions/$FSROOT";
        fsType = "btrfs";
        options = "subvol=root";
      };

      fileSystems."/home" = {
        label = "home";
        device = "/dev/partitions/$FSROOT";
        fsType = "btrfs";
        options = "subvol=home";
      };

Finally, clean up and you should be ready to reboot into your new system:

    umount /mnt/{home,boot,}
    swapoff /dev/partitions/swap
    vgchange -an
    cryptsetup luksClose "$LUKSROOT"