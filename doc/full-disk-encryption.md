Full Disk Encryption Setup
==========================

This manual describes an installation procedure of Ubuntu 20.04 LTS with full
disk encryption.

This manual is based on [Full_Disk_Encryption_Howto_2019][FDE] guide.

[FDE]: https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019

In this document a system drive is defined as `/dev/sda` device. The system drive
partitioning:

| Partition # | Content                             |
| ----------- | ----------------------------------- |
| 1           | EFI system partition                |
| 2           | Encrypted (LUKS) `/boot` filesystem |
| 3           | Encrypted (LUKS) LVM                |

The system drive LVM logical volumes:

*   root filesystem
*   `/home` filesystem
*   swap

The setup instructions:

1.  Boot in UEFI mode from installation media and choose "Try Ubuntu":

    ![Try Ubuntu](figures/try-ubuntu.png)

2.  Open terminal application and switch to root session:

    ```sh
    sudo -i
    ```

3.  Wipe the internal drive with random data:

    ```sh
    shred \
        --verbose \
        --iterations=3 \
        --random-source=/dev/urandom \
        /dev/sda
    ```

4.  Partition the internal drive according to the following scheme:

    | Part. # | Name    | Type                 | Size                 |
    | ------- | ------- | -------------------- | -------------------- |
    | 1       | esp     | EFI system partition | 1G                   |
    | 2       | boot    | Linux LUKS           | 1G                   |
    | 3       | primary | Linux LUKS           | Remaining free space |

    ```sh
    sgdisk --zap-all /dev/sda

    sgdisk --new=0:0:+1G /dev/sda
    sgdisk --new=0:0:+1G /dev/sda
    sgdisk --new=0:0:0 /dev/sda

    sgdisk --change-name=1:esp /dev/sda
    sgdisk --change-name=2:boot /dev/sda
    sgdisk --change-name=3:primary /dev/sda

    sgdisk --typecode=1:ef00 /dev/sda
    sgdisk --typecode=2:8309 /dev/sda
    sgdisk --typecode=3:8309 /dev/sda
    ```

5.  Initialize LUKS partitions according to the following scheme:

    | Partition # | Device      | LUKS version |
    | ----------- | ----------- | ------------ |
    | 2           | `/dev/sda2` | 1            |
    | 3           | `/dev/sda3` | 2            |

    ```sh
    cryptsetup luksFormat --type=luks1 /dev/sda2
    cryptsetup luksFormat --type=luks2 /dev/sda3
    ```

6.  Open the initialized LUKS partitions as the following devices:

    | Partition # | Device      | LUKS device           |
    | ----------- | ----------- | --------------------- |
    | 2           | `/dev/sda2` | `/dev/mapper/boot`    |
    | 3           | `/dev/sda3` | `/dev/mapper/primary` |

    ```sh
    cryptsetup open /dev/sda2 boot
    cryptsetup open /dev/sda3 primary
    ```

7.  Initialize `esp` and `boot` partitions according to the following scheme:

    | Device             | FS    |
    | ------------------ | ----- |
    | `/dev/sdb1`        | FAT32 |
    | `/dev/mapper/boot` | ext4  |

    ```sh
    mkfs.vfat -F 32 /dev/sda1
    mkfs.ext4 /dev/mapper/boot
    ```

8.  Initialize `primary` partition (`/dev/mapper/primary` device) with
    `vg-primary` LVM volume group according to the following scheme:

    | Logical Volume | Size | FS    |
    | -------------- | ---- | ----- |
    | root           | 10G  | XFS   |
    | home           | 20G  | XFS   |
    | swap           | 8G   | swap  |

    ```sh
    pvcreate /dev/mapper/primary

    vgcreate vg-primary /dev/mapper/primary

    lvcreate --name=root --size=10G vg-primary
    lvcreate --name=home --size=20G vg-primary
    lvcreate --name=swap --size=8G vg-primary

    mkfs.xfs /dev/vg-primary/root
    mkfs.xfs /dev/vg-primary/home
    mkswap /dev/vg-primary/swap
    ```

9.  Start Ubuntu installation:

    ![Install Ubuntu 20.04.2.0 LTS](figures/install-ubuntu.png)

10. Choose the preferred language:

    ![Preferred Language](figures/language.png)

11. Choose the keyboard layout:

    ![Keyboard layout](figures/keyboard-layout.png)

12. Choose the options for updates and other software:

    ![Updates and other software](figures/updates-and-other-software.png)

13. In "Installation type" window choose "Something else":

    ![Installation type](figures/installation-type.png)

14. Configure the following options:

    | Device                               | Type  | Mount Point | Format |
    | ------------------------------------ | ----- | ----------- | ------ |
    | /dev/mapper/boot                     | ext4  | /boot       | no     |
    | /dev/mapper/vg--primary-home         | xfs   | /home       | no     |
    | /dev/mapper/vg--primary-root         | xfs   | /           | no     |
    | /dev/mapper/vg--primary-swap         | swap  |             | no     |
    | /dev/sda1                            | efi   |             | no     |

    Device for boot loader installation: `/dev/sda`.

    ![Partitioner](figures/partitioner.png)

    ![Do you want to return to the partitioner](figures/return-to-partitioner.png)

    ![Whrite the changes to disks?](figures/write-changes.png)

15. Configure timezone:

    ![Where are you?](figures/where-are-you.png)

16. Configure hostname and user account:

    ![Who are you?](figures/who-are-you.png)

17. When installation starts, enable `grub-mkconfig` and `grub-install` to check
    for encrypted disks.

    **Warning.** This has to be done before the installer reaches bootloader
    installation stage.

    ![Installation Progress](figures/progress.png)

    Run the following commands in terminal:

    ```sh
    while [ ! -d /target/etc/default/grub.d ]; do
        sleep 1
    done; \
    echo "GRUB_ENABLE_CRYPTODISK=y" > /target/etc/default/grub.d/cryptodisk.cfg
    ```

18. When the installation is complete, select "Continue Testing":

    ![Installation Complete](figures/installation-complete.png)

19. Chroot to installed system:

    ```sh
    mount /dev/vg-primary/root /target
    for path in proc sys dev etc/resolv.conf; do
        mount --rbind /$path /target/$path
    done
    chroot /target
    mount --all
    ```

20. Configure encrypted devices to be unlocked automatically.

    Ensure that `cryptsetup-initramfs` package is installed:

    ```sh
    apt install cryptsetup-initramfs
    ```

    Set the following option in `/etc/cryptsetup-initramfs/conf-hook`:

    ```sh
    KEYFILE_PATTERN="/etc/luks/*.key"
    ```

    Create file `/etc/initramfs-tools/conf.d/umask` with the following content:

    ```sh
    UMASK=0077
    ```

    Create and configure keys:

    ```sh
    mkdir --mode=500 /etc/luks
    dd if=/dev/urandom of=/etc/luks/boot.key bs=512 count=1
    dd if=/dev/urandom of=/etc/luks/primary.key bs=512 count=1
    chmod 400 /etc/luks/*.key

    cryptsetup luksAddKey /dev/sda2 /etc/luks/boot.key
    cryptsetup luksAddKey /dev/sda3 /etc/luks/primary.key

    BOOT_UUID=$(blkid --match-tag=UUID --output=value /dev/sda2)
    echo "boot UUID=$BOOT_UUID /etc/luks/boot.key luks" > /etc/crypttab

    PRIMARY_UUID=$(blkid --match-tag=UUID --output=value /dev/sda3)
    echo "primary UUID=$PRIMARY_UUID /etc/luks/primary.key luks" >> /etc/crypttab

    update-initramfs -u -k all
    ```

21. Reboot.
