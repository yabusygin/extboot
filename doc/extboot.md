Extboot Setup
=============

This manual describes the extboot installation and configuration.

A special loop device (`/dev/loop78`) with staging ESP is created. When OS is
booted up this device is mounted to `/boot/efi` instead of the external drive
ESP. When EFI applications (shim or GRUB) are updated, they are first updated on
the staging ESP. When the external drive is connected, its ESP is synced with
the staging ESP.

It's assumed that internal drive is partitioned according to the
`${project_root}/doc/full-disk-encryption.md` manual. Other partitioning schemes
may be used, but the following setup instructions have to be adjusted
accordingly.

The setup instructions:

1.  Install `extboot` dependencies:

    ```sh
    apt install rsync
    ```

2.  Initialize `extboot` directory:

    ```sh
    mkdir --parents /var/lib/extboot/mnt/esp
    mkdir /var/lib/extboot/mnt/recovery
    ```

3.  Initialize backing file for the staging ESP:

    ```sh
    dd if=/dev/zero of=/var/lib/extboot/esp.img bs=1M count=1024
    mkfs.vfat -F 32 /var/lib/extboot/esp.img
    ```

4.  Connect the external drive.

5.  Initialize the external drive according to the following scheme:

    | Partition # | Name     | Type                 | Size                 |
    | ----------- | -------- | -------------------- | -------------------- |
    | 1           | esp      | EFI system partition | 1G                   |
    | 2           | recovery | Linux filesystem     | Remaining free space |

    | Partition # | Device      | FS    | FS Label    |
    | ----------- | ----------- | ----- | ----------- |
    | 1           | `/dev/sdb1` | FAT32 | EB-ESP      |
    | 2           | `/dev/sdb2` | XFS   | EB-RECOVERY |

    ```sh
    sgdisk --zap-all /dev/sdb

    sgdisk --new=0:0:+1G /dev/sdb
    sgdisk --new=0:0:0 /dev/sdb

    sgdisk --change-name=1:esp /dev/sdb
    sgdisk --change-name=2:recovery /dev/sdb

    sgdisk --typecode=1:ef00 /dev/sdb
    sgdisk --typecode=2:8300 /dev/sdb

    mkfs.vfat -F 32 -n EB-ESP /dev/sdb1
    mkfs.xfs -L EB-RECOVERY /dev/sdb2
    ```

6.  Synchronize the staging, the extboot and the system ESPs:

    Initialize the extboot ESP (on external drive):

    ```sh
    # Fix loop device mount. See the note bellow.
    losetup /dev/loop78 /var/lib/extboot/esp.img
    losetup --detach /dev/loop78

    mount \
        --types=vfat \
        --options='umask=0077' \
        /var/lib/extboot/esp.img \
        /var/lib/extboot/mnt/esp
    rsync --archive --delete /boot/efi/ /var/lib/extboot/mnt/esp
    umount /var/lib/extboot/mnt/esp

    mount \
        --types=vfat \
        --options='umask=0077' \
        LABEL=EB-ESP \
        /var/lib/extboot/mnt/esp
    rsync --archive --delete /boot/efi/ /var/lib/extboot/mnt/esp
    umount /var/lib/extboot/mnt/esp
    ```

    **Note.** Without running the specified above `losetup` commands mounting of
    `/var/lib/extboot/esp.img` image fails with the following error message:

    ```
    mount: /boot/efi: failed to setup loop device for /var/lib/extboot/esp.img.
    ```

7.  Configure mounts.

    Set `/etc/fstab` content as specified in `${project_root}/config/fstab`.

    Create file `/etc/systemd/system/extboot-fix-mount.service` with contents
    from `${project_root}/config/systemd-service/extboot-fix-mount.service`.

    **Warning.** Without the above service mounting of
    `/var/lib/extboot/esp.img` image fails with the following error message:

    ```
    mount: /boot/efi: failed to setup loop device for /var/lib/extboot/esp.img.
    ```

8.  Configure synchronization of the extboot and the staging ESPs.

    Create file `/etc/systemd/system/extboot-sync.service` with contents from
    `${project_root}/config/systemd-service/extboot-sync.service`.

    Create file `/etc/udev/rules.d/99-extboot.rules` with contents from
    `${project_root}/config/udev/99-extboot.rules`.

9.  Disable UEFI boot variables updates.

    ```sh
    echo 'set grub2/update_nvram false' | debconf-communicate
    ```

    To fix the issue #13 run the following command:

    ```sh
    dpkg-divert \
        --add \
        --divert /usr/lib/grub/grub-multi-install.distrib \
        --rename \
        /usr/lib/grub/grub-multi-install
    ```

    and create wrapper script `/usr/lib/grub/grub-multi-install` with contents from
    `${project_root}/wrapper/grub-multi-install`.

    ```sh
    chmod +x /usr/lib/grub/grub-multi-install
    ```

10. Clean ESP on internal drive:

    ```sh
    umount /dev/sda1
    sgdisk --change-name=1:extboot-reserved-1 /dev/sda
    sgdisk --typecode=1:8da63339-0007-60c0-c436-083ac8230908 /dev/sda
    shred \
        --verbose \
        --iterations=3 \
        --random-source=/dev/urandom \
        /dev/sda1
    ```

11. Update UEFI boot variable.

    List all boot variables and their labels:

    ```sh
    efibootmgr
    ```

    Find the variable with label `ubuntu` and assign its number to shell
    variable `BOOT_NUM`. For example for the entry `Boot0002* ubuntu` use the
    following assignment:

    ```sh
    BOOT_NUM=0002
    ```

    Update boot variables to point to the extboot ESP:

    ```sh
    efibootmgr --delete-bootnum --bootnum=${BOOT_NUM}
    efibootmgr \
        --create \
        --label=ubuntu \
        --disk=/dev/sdb \
        --part=1 \
        --loader='\EFI\ubuntu\shimx64.efi'
    ```

12. Reboot.
