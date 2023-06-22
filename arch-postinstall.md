# Post Install

## TODO
- Snapper

- Periodic SSD TRIM: https://wiki.archlinux.org/title/Solid_state_drive#Periodic_TRIM

- BTRFS Balance: https://btrfs.readthedocs.io/en/latest/Balance.html

## Setup the network
```sh
systemctl enable NetworkManager.service # Casing is important
systemctl start NetworkManager.service
ping archlinux.org
```

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

### Enable Parallel Downloads
https://wiki.archlinux.org/title/Pacman#Enabling_parallel_downloads

Uncomment:
- `ParallelDownloads`
- `Color`
- `ILoveCandy`
- `mutilib`
```sh
nano /etc/pacman.conf
```

## Use `zsh`
```sh
TODO
```

## Install nvidia Drivers
```sh
pacman -S nvidia
```

## Install DE with Wayland
```sh
pacman -S plasma plasma-wayland-session sddm
systemctl enable sddm.service
systemctl start sddm.service

# TODO install nvidia_DRM
```

## Look into Plymouth

## Secure the DNS
https://wiki.archlinux.org/title/Domain_name_resolution#Privacy_and_security

## Setup SSHD and configure


## GRUB themes

## Install linux-lts kernel as a backup option and setup grub multi option

## paccache hooks
https://wiki.archlinux.org/title/Pacman#Cleaning_the_package_cache

## CPU Scaling
https://wiki.archlinux.org/title/CPU_frequency_scaling
## My Applications
- Steam
- Firefox
- KeepassXC


# References

- https://wiki.archlinux.org/title/General_recommendations