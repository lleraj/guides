# Encrypted Arch on Btrfs subvolumes incl. /boot

- [Encrypted Arch on Btrfs subvolumes incl. /boot](#encrypted-arch-on-btrfs-subvolumes-incl-boot)
  - [Create Partitions](#create-partitions)
    - [Create /boot Filesystem](#create-boot-filesystem)
    - [Create /boot/efi Filesystem](#create-bootefi-filesystem)
    - [Create / Filesystem](#create--filesystem)
  - [Create Subvolumes](#create-subvolumes)
    - [Mount root partition](#mount-root-partition)
    - [Create subvolumes](#create-subvolumes-1)
    - [Mount @ subvolume as root partition](#mount--subvolume-as-root-partition)
    - [Mount @home subvolume](#mount-home-subvolume)
    - [Mount @snapshots subvolume](#mount-snapshots-subvolume)
  - [Mount remaining partitions](#mount-remaining-partitions)
    - [/boot](#boot)
    - [/boot/efi](#bootefi)
  - [Install Arch](#install-arch)
    - [Install required packages](#install-required-packages)
    - [Generate fstab](#generate-fstab)
    - [Chroot into new system](#chroot-into-new-system)
    - [Generate Keyfile for automatic unlocking](#generate-keyfile-for-automatic-unlocking)
    - [Add crypttab](#add-crypttab)
    - [Add mkinitcpio config](#add-mkinitcpio-config)
    - [Add GRUB config](#add-grub-config)

---

## Create Partitions

```sh
gdisk /dev/sda
```

| Path      | Size      | Type | Format |
| --------- | --------- | ---- | ------ |
| /boot     | 550M      | 8309 | luks1  |
| /boot/efi | 200M      | EF00 | Fat32  |
| /         | remaining | 8309 | luks2  |

### Create /boot Filesystem

```sh
cryptsetup luksFormat --type luks1 /dev/sda1
```

```sh
cryptsetup luksOpen /dev/sda1 cryptboot
```

```sh
mkfs.ext4 /dev/mapper/cryptboot
```

### Create /boot/efi Filesystem

```sh
mkfs.fat -F32 /dev/sda2
```

### Create / Filesystem

```sh
cryptsetup luksFormat /dev/sda3
```

```sh
cryptsetup luksOpen /dev/sda3 cryptroot
```

```sh
mkfs.btrfs /dev/mapper/cryptroot
```

## Create Subvolumes

| Name       | Path        |
| ---------- | ----------- |
| @          | /           |
| @home      | /home       |
| @snapshots | /.snapshots |

### Mount root partition

```sh
mount /dev/mapper/cryptroot /mnt
```

### Create subvolumes

```sh
btrfs subvolume create /mnt/@
```

```sh
btrfs subvolume create /mnt/@home
```

```sh
btrfs subvolume create /mnt/@snapshots
```

### Mount @ subvolume as root partition

```sh
umount /mnt
```

```sh
mount -o compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
```

### Mount @home subvolume

```sh
mkdir /mnt/home
```

```shh
mount -o compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
```

### Mount @snapshots subvolume

```sh
mkdir /mnt/.snapshots
```

```sh
mount -o compress=zstd,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
```

## Mount remaining partitions

### /boot

```sh
mkdir /mnt/boot
```

```sh
mount /dev/mapper/cryptboot /mnt/boot
```

### /boot/efi

```sh
mkdir /mnt/boot/efi
```

```sh
mount /dev/sda2 /mnt/boot/efi
```

## Install Arch

### Install required packages

```sh
pacstrap /mnt base linux btrfs-progs grub efibootmgr vim
```

### Generate fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot into new system

```sh
arch-chroot /mnt
```

### Generate Keyfile for automatic unlocking

```sh
dd bs=512 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
```

```sh
chmod 600 /crypto_keyfile.bbin
chmod 600 /boot/initramfs-linux*
```

```sh
cryptsetup luksAddkey /dev/sda1 /crypto_keyfile.bin
cryptsetup luksAddKey /dev/sda3 /crypto_keyfile.bin
```

### Add crypttab

```sh
cryptboot /dev/sda1 /crypto_keyfile.bin luks
```

### Add mkinitcpio config

```sh
BINARIES=(/usr/bin/btrfs)
FILES=(/crypto_keyfile.bin)
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt filesystems fsck)
```

### Add GRUB config

```sh
GRUB_CMDLINE_LINUX="cryptdevice=UUID=$DEVICEUUID:cryptroot root=/dev/mapper/cryptroot"
```
