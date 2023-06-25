# Post Install

## TODO
- Periodic SSD TRIM: https://wiki.archlinux.org/title/Solid_state_drive#Periodic_TRIM

- BTRFS Balance: https://btrfs.readthedocs.io/en/latest/Balance.html

## Enable network manager
```sh
systemctl enable NetworkManager.service # Casing is important
systemctl start NetworkManager.service
```

## Install yay

## Add normal user
```sh
useradd -m -G wheel -s /bin/bash dem1se
passwd dem1se
nano /etc/sudoers
# %wheel      ALL=(ALL:ALL) ALL
```

## Pacman configuration
### Use the latest and fastest mirrors automatically
https://wiki.archlinux.org/title/reflector
```sh
pacman -S reflector

# I am happy with the default configuration
# systemctl edit reflector.timer
systemctl enable reflector.timer
```

### Enable Parallel Downloads and `multilib`

Uncomment the following lines:
- `ParallelDownloads`
- `Color`
- `mutilib` - for installing Steam and other gaming things
```sh
nano /etc/pacman.conf
```

## Use `zsh`

## Install Nvidia Propriertary Drivers
```sh
pacman -S nvidia nvidia-utils nvidia-settings
```

run nviida-xconfig later

## Desktop Environment
Also need to use `sddm-git` AUR package, since 20.x version supports Wayland, while official release in the repo was 19.x at the time.
```sh
pacman -S plasma plasma-wayland-session sddm

pacman -S kde-applications
``` 

`/etc/modprobe.d/nvidia.conf`
```
options nvidia-drm modeset=1 
options nvidia NVreg_UsePageAttributeTable=1
```

`mkinitcpio.conf`
```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

`/etc/default/grub.conf`
```
GRUB_GFXMODE=1920x1080x32,1024x768x32,auto
GRUB_GFXPAYLOAD_LINUX=keep
```
## Setup `snapper`

Based on: https://wiki.archlinux.org/title/Snapper#Configuration_of_snapper_and_mount_point

```sh
pacman -S snapper grub-btrfs
yay -S btrfs-assistant

# Needed to let snapper make its snapshots folder
umount /.snapshots
rm -r /.snapshots

# create the config
snapper -c root create-config /

# delete the subvolume that snapper created
btrfs sub delete /.snapshots

# make the folder manually
mkdir /.snapshots

# create mount the original subvol again
mount -a # based on fstab
```

Following this, use `btrfs-assistant` to make the first snapshot, and enable systemd units for timeline and cleanup

## Secure the DNS
https://wiki.archlinux.org/title/Domain_name_resolution#Privacy_and_security

## Enable bluetooth
```sh
pacman -S bluez bluez-utils
systemctl enable bluetooth.service
systemctl start bluetooth.service
```



## Setup SSHD and configure it

## GRUB theme

## pacman cache hooks
https://wiki.archlinux.org/title/Pacman#Cleaning_the_package_cache

## CPU Scaling
https://wiki.archlinux.org/title/CPU_frequency_scaling
## Some Applications
- Steam
- Firefox
- KeepassXC


## Other Things I did
- Install PulseAudio
- Wayland everything
- Google drive setup
- SDDM multimonitor
- libratbag
- Plymouth
# References

- https://wiki.archlinux.org/title/General_recommendations
