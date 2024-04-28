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

## Delete old FS

If the drive isn't completely fresh. This is a single drive, arch only install.

```sh
wipefs -af /dev/nvme0n1
sgdisk --zap-all --clear /dev/nvme0n1
partprobe /dev/nvme0n1
```
## Change to optimal LBA format for disk
```sh
nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"
nvme format --lbaf=1 /dev/nvme0n1
```

## Partition the disks:

I'm using a fresh 1000G nvme drive. Only using 600G for the root, since I plan to use the rest for a gaming VM later.

| Size | Type    | Partition |
|------|---------|-----------|
| 500M | EFI (1) | /efi      |
| 500M | ext4    | /boot     |
| 600G | btrfs   | /         |

```sh
fdisk /dev/nvme0n1
# g n +500M t 1 n +500M n +600G w
mkfs.fat -n ESP /dev/nvme0n1p1 # Create FS for EFI
```

## Create LUKS volumes and mkfs inside them

### For the boot partition:
```sh
cryptsetup luksFormat --type luks1 /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 boot_crypt
mkfs.ext4 -L boot /dev/mapper/boot_crypt
```

### For the root parition:

#### Generate the randomtext key for the root:
```sh
dd bs=4096 count=1 if=/dev/random of=root_keyfile.bin iflag=fullblock # be careful not to lose this file by restarting the install env
chmod 600 root_keyfile.bin # remove read perms from others
```

#### Do the luks and mkfs steps

Letting the keyfile take the first key slot, this way the normal flow is faster.
```sh
cryptsetup luksFormat --type luks2 /dev/nvme0n1p3 root_keyfile.bin
# Adding a backup pass phrase. This will be in slot 1, --key-file switch is to authenticate with current method before adding new one.
cryptsetup luksAddKey --key-file=root_keyfile.bin /dev/nvme0n1p3

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
btrfs subvolume create /mnt/@swap      # to be mounted at /swap
umount /mnt
```

#### Mount the top subvolumes:
```sh
export sv_opts="noatime,compress=zstd:1,space_cache=v2"

mount -o $sv_opts,subvol=@ /dev/mapper/root_crypt /mnt

mkdir /mnt/home
mount -o $sv_opts,subvol=@home /dev/mapper/root_crypt /mnt/home

mkdir /mnt/.snapshots
mount -o $sv_opts,subvol=@snapshots /dev/mapper/root_crypt /mnt/.snapshots

mkdir -p /mnt/var/log
mount -o $sv_opts,subvol=@var_log /dev/mapper/root_crypt /mnt/var/log

mkdir /mnt/swap
mount -o $sv_opts,subvol=@swap /dev/mapper/root_crypt /mnt/swap
```

#### Create exclusion subvolumes
```sh
mkdir -p /mnt/var/cache/pacman/
btrfs subvolume create /mnt/var/cache/pacman/pkg
btrfs subvolume create /mnt/var/abs
btrfs subvolume create /mnt/var/tmp
btrfs subvolume create /mnt/srv
```

### Mount /efi and /boot before generating fstab
```sh
mkdir /mnt/efi
mount /dev/nvme0n1p1 /mnt/efi

mkdir /mnt/boot
mount /dev/mapper/boot_crypt /mnt/boot

mkdir -p /mnt/etc/keys
cp root_keyfile.bin /mnt/etc/keys
```

## Pacstrap on /mnt
```sh
pacstrap /mnt base linux linux-firmware btrfs-progs cryptsetup networkmanager man-db man-pages amd-ucode neovim reflector sudo
```

## Generate fstab
```sh
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # verify and edit the entries if needed
```

## Chroot into the new install
```sh
arch-chroot /mnt
```

## Set timezone, locale, and setup hostnames
```sh
ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
nvim /etc/locale.gen # uncomment en_US and en_IN lines
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "cardinal" > /etc/hostname
echo '127.0.0.1	localhost
::1		localhost
127.0.1.1	cardinal.localdomain	cardinal' > /etc/hosts
```

## Set root password
```sh
passwd
```

## Generate keyfile for bootloader to decrypt `/boot` partition automatically at late userspace

We needed to unlock the boot partition for the second time in late userspace. When the kernel mounts all the partitions in `/etc/fstab`, it would need to unlock the the boot partition luks volume, and promt for the password which is undesirable. This fix makes it so that the kernel has a key file that it can use to unlock the volume. I am unsure if you can make the system prompt for password when it attempts to unlock the boot partition for the second time, but it was not happening in my case by default, so this not just convenient, but also required from what I know.

It is worth noting that this is different from the reason other setups use keyfile for the /boot: in the setups where the /boot is within the (LUKS1 encrypted) root partition, the encrypted volume is unlocked when GRUB tries to access /boot, and then for a second time when the kernel tries to unlock the root. In that case, there is no need to use crypttab.

The bootloader will still prompt for the passphrase once, before reaching the GRUB menu, which can be solved by using TPMs.

```sh
dd bs=4096 count=1 if=/dev/random of=/etc/keys/boot_keyfile.bin iflag=fullblock
chmod 600 /etc/keys/boot_keyfile.bin
chmod 600 /boot/initramfs-linux*
cryptsetup luksAddKey /dev/nvme0n1p2 /etc/keys/boot_keyfile.bin
```

### Add /boot to cryptab to auto unlock at late userspace

```sh
echo "# Mount /dev/nvme0n1p2 (/boot parition) to /dev/mapper/boot_crypt using keyfile at /keys
boot_crypt      /dev/nvme0n1p2  /etc/keys/boot_keyfile.bin" >> /etc/crypttab

cat /etc/crypttab # verify
```

fstab references this unlocked luks volume (boot_crypt) for mounting (by uuid).

## Setting up initramfs

This will be used by the kernel to setup the temporary ramfs, which can then switch to the real root.

```sh
nvim /etc/mkinitcpio.conf
```

### Things to edit in the mkinitcpio.conf
```sh
BINARIES=(/usr/bin/btrfs)
FILES=(/etc/keys/root_keyfile.bin)
HOOKS=(base udev keyboard autodetect microcode keymap consolefont modconf block encrypt filesystems fsck)
```

```sh
mkinitcpio -P
```

## Install and configure GRUB
```sh
# Install grub
pacman -S grub efibootmgr

lsblk -f # or blkid

# TODO add resume option
# GRUB_ENABLE_CRYPTODISK=y
# GRUB_PRELOAD_MODULES="part_gpt part_msdos luks" # add luks
# Not sure if LABEL based methods worked properly, I didn't verify the grub.cfg
# GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=LABEL=root:root_crypt:allow-discards cryptkey=rootfs:/etc/keys/root_keyfile.bin rootflags=subvol=@"
# GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=03d096e4-f4e0-4ce4-b690-09c94421ad85:root_crypt:allow-discards cryptkey=rootfs:/etc/keys/root_keyfile.bin rootflags=subvol=@"
nvim /etc/default/grub

grub-mkconfig -o /boot/grub/grub.cfg
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=ARCH-GRUB
```

## Setup Swapfile
```sh
btrfs filesystem mkswapfile --size 16G /swap/swapfile
swapon /swap/swapfile
echo "/swap/swapfile none swap sw 0 0" >> /etc/fstab
```

### (Optional) Enable Hibernation into swapfile

https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file

```sh
btrfs inspect-internal map-swapfile -r /swap/swapfile

# Add `resume` to HOOKS at: HOOKS=( ...filesystems resume fsck)
nvim /etc/mkinitcpio.conf
mkinitcpio -P

nvim /etc/default/grub
# add resume=UUID=swap_device_uuid and resume_offset=swap_file_offset, that you found in the first line above
grub-mkconfig -o /boot/grub/grub.cfg
```

## Reboot into install
```sh
reboot

# Check for errors
systemctl --failed
journalctl -p 3 -xb
```

# Desktop Setup After Install

*See arch-postinstall.md*


# Other Neat Features TODO
- Suspend Support: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation

- Secure boot w/ TPM: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot

- Remote unlocking of root: https://wiki.archlinux.org/title/Dm-crypt/Specialties#Remote_unlocking_of_root_(or_other)_partition

# References

- https://wiki.archlinux.org/title/Installation_guide

- https://wiki.archlinux.org/title/Arch_boot_process

- https://forum.level1techs.com/t/gkh-threadripper-3970x-setup-notes/156330

- https://gitlab.com/Thawn/arch-encrypted

- https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file

- https://wiki.archlinux.org/title/dm-crypt/Device_encryption#Backup_and_restore

- https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout

# Debugging helper snippets
## remount all
For when you need to remount everything to the archiso live env after rebooting and having to debug something.

```sh
cryptsetup open /dev/nvme0n1p3 root_crypt
mount -o subvol=@ /dev/mapper/root_crypt /mnt
mount -o subvol=@home /dev/mapper/root_crypt /mnt/home
mount -o subvol=@snapshots /dev/mapper/root_crypt /mnt/.snapshots
mount -o subvol=@var_log /dev/mapper/root_crypt /mnt/var/log

mount /dev/nvme0n1p1 /mnt/efi

cryptsetup open /dev/nvme0n1p2 boot_crypt
mount /dev/mapper/boot_crypt /mnt/boot
```

```sh
cryptsetup open /dev/nvme0n1p3 root_crypt
mount -o subvol=@ /dev/mapper/root_crypt /mnt
mount -o subvol=@home /dev/mapper/root_crypt /mnt/home
mount -o subvol=@snapshots /dev/mapper/root_crypt /mnt/.snapshots
mount -o subvol=@var_log /dev/mapper/root_crypt /mnt/var/log

mount /dev/nvme0n1p1 /mnt/efi
mount /dev/nvme0n1p2 /mnt/boot
```