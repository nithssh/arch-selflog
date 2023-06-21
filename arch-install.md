# Arch Install Personal Log

The idea is to have a LUKS2 encrypted root, and a separate LUKS1 encrypted /boot parition that GRUB can open.

GRUB should be able to open the LUKS1 volume, meaning it should have the boot keyfile embedded in it. This is done with the `/etc/default/grub` file and `grub-install` command. Enabling cryptodisk support in GRUB.

initramfs should be configured to allow the encrypted root to be opened. So the root keyfile should be embedded in that though the `/etc/mkinitcpio.conf` modules, files, and kernel parameters. Personal note: `crypttab` doesn't apply here, since it's for automounting external/secondary devices during boot, and isn't for root explicitly.

## Ensure time is correct, and prepare SSHD
```sh
timedatectl # usually redundant
ip a
passwd
systemctl status sshd
```

## Change to optimal LBA format for disk
```sh
nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"
nvme format --lbaf=1 /dev/nvme0n1
```

## Partition the disks:

I'm using a 1000G nvme drive. Only using 600G for the root, since I plan to use the rest for a gaming VM later.

| Size | Type    | Partition |
|------|---------|-----------|
| 500M | EFI (1) | /esp      |
| 500M | ext4    | /boot     |
| 600G | btrfs   | /         |

```sh
disk /dev/nvme0n1
mkfs.fat /dev/nvme0n1p1 # Create FS for EFI
```

## Create LUKS volumes and mkfs inside them

### For the boot partition:
```sh
cryptsetup luksFormat --type luks1 /dev/nvme0n1p2
cryptsetup open /dev/nvmeon1p2 boot_crypt
mkfs.ext4 -L boot /dev/mapper/boot_crypt
```

### For the root parition:

#### Generate the randomtext key for the root:
```sh
dd bs=4096 count=1 if=/dev/random of=root_keyfile.bin iflag=fullblock
```

#### Do the luks and mkfs steps
```sh
cryptsetup luksFormat --type luks2 /dev/nvme0n1p3 root_keyfile.bin
cryptsetup open --key-file=root_keyfile.bin /dev/nvme0n1p3 root_crypt
mkfs.btrfs -L root /dev/mapper/root_crypt
```

#### Create the btrfs subvolumes
```sh
mount /dev/mapper/root_crypt /mnt
btrfs subvolume create /mnt/@          # to be mounted at /
btrfs subvolume create /mnt/@home      # to be mounted at /home
btrfs subvolume create /mnt/@snapshots # to be mounted at /.snapshots
btrfs subvolume create /mnt/@var_log   # to be mounted at /var/log
umount /mnt
```

#### Mount the top subvolumes:
```sh
mount -o compress=zstd,subvol=@ /dev/mapper/root_crypt /mnt
mkdir /mnt/home
mount -o compress=zstd,subvol=@home /dev/mapper/root_crypt /mnt/home
mkdir /mnt/.snapshots
mount -o compress=zstd,subvol=@snapshots /dev/mapper/root_crypt /mnt/.snapshots
mkdir -p /mnt/var/log
mount -o compress=zstd,subvol=@var_log /dev/mapper/root_crypt /mnt/var/log
```

### Mount /esp and /boot before generating fstab
```sh
mkdir /mnt/esp
mount /dev/sda1 /mnt/esp

mkdir mnt/boot
mount /dev/mapper/boot_crypt /mnt/boot
```

## Generate fstab
```sh
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # verify and edit the entries if needed
```

## Pacstrap on mnt:
```sh
pacstrap /mnt base linux linux-firmware btrfs-progs
pacstrap /mnt man-db man-pages bash-completion amd-ucode nano
```

## Chroot into the new install
```sh
arch-chroot /mnt
```

## Set timezone, locale, and setup hostnames
```sh
ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
nano /etc/locale.gen # uncomment en_US line
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "cardinal" > /etc/hostname
echo '127.0.0.1	localhost
::1		localhost
127.0.1.1	cardinal.localdomain	cardinal' > /etc/hosts
```

## enable this ig? probably don't need to _rn_

```sh
systemctl enable systemd-networkd systemd-resolved
```

## Set root password
```sh
passwd
```

## Generate key for bootloader to decrypt boot partition automatically
```sh
dd bs=4096 count=1 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
chmod 600 /crypto_keyfile.bin
chmod 600 /boot/initramfs-linux*
cryptsetup luksAddKey /dev/nvme0n1p2 /crypto_keyfile.bin

# Add a pass phrase now just in case I can't boot with key
cryptsetup luksAddKey ==key-file=/boot/root_keyfile.bin /dev/nvme0n1p3
```

## Setup initramfs

This will be run by the bootloader to then open and switch to the real root.

```sh
nano /etc/mkinitcpio.conf
```

### Things to edit in the mkinitcpio.conf
```sh
BINARIES=(/usr/bin/btrfs)
FILES=(/crypto_keyfile.bin)
# Look into alternate orderings from community
HOOKS=(base udev keyboard autodetect keymap consolefont modconf block encrypt filesystems fsck)
```

```sh
mkinitcpio -P
```

## Install and configure GRUB
```sh
# Install grub
pacman -S grub efibootmgr

# Set the crypto disk option to 'y' and also edit the kernel parameters
nano /etc/default/grub

grub-mkconfig -o /boot/grub/grub.cfg
grub-install --target=x86_64-efi --efi-directory=/esp --bootloader-id=ARCH-GRUB
```

```sh
uuid=$( blkid -o value /dev/nvme0n1 | head -n 1 )
```



# Post install TODO
- Secure boot w/ TPM: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot

- Suspend Support: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation



# References

[1] https://wiki.archlinux.org/title/Installation_guide

[2] https://gitlab.com/Thawn/arch-encrypted

[3] https://wiki.archlinux.org/title/Btrfs

[4] https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file

[5] https://forum.level1techs.com/t/gkh-threadripper-3970x-setup-notes/156330

[5] https://wiki.archlinux.org/title/dm-crypt/Device_encryption#Keyfiles

[6] https://wiki.archlinux.org/title/dm-crypt/Device_encryption#Backup_and_restore

[7] https://access.redhat.com/solutions/230993

[8] https://linuxconfig.org/introduction-to-crypttab-with-examples

[9] https://wiki.archlinux.org/title/Arch_boot_process

[10] https://unix.stackexchange.com/questions/384494/efi-partition-vs-boot-partition

[11] https://www.reddit.com/r/btrfs/comments/l1w4ye/which_dirs_to_exclude_from_root_snapshot/

[12] https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout