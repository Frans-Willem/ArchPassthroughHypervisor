# Archlinux installation
## Basic system setup
* Boot from CD/USB/DVD, either using EFI or 
* Check what your primary drive is (/dev/sd?)
  ```
  lsblk
  ```
  From now on I will be using /dev/sda, if your primary drive is different, adjust accordingly.
* Partition your drive
  * Start
    ```sh
    cgdisk
    ```
  * Create an EFI partition: 512MB, type EF00, label "EFI Boot"
  * Create an LVM partition: rest, type 8E00, label "Linux LVM"
  * Write
  * Quit
* Create a FAT32 filesystem on the EFI drive first:
  ```
  mkfs.fat -F32 /dev/sda1
  ```
* Create an LVM Physical Volume on the LVM partition:
  ```
  pvcreate /dev/sda2
  ```
* Create a volume group on that physical volume
  ```
  vgcreate vg-Main /dev/sda2
  ```
* Create boot, swap, and root "Logical volumes"
  ```
  lvcreate --name boot --size 1GB vg-Main
  lvcreate --name swap -- size 32GB vg-Main
  lvcreate --name root --size 20GB vg-Main
  ```
* Create the filesystems
  ```
  mkfs.ext2 /dev/vg-Main/boot
  mkfs.ext4 /dev/vg-Main/root
  mkswap /dev/vg-Main/swap
  ```
* Mount the filesystems, create the mountpoints, and activate the swap partition
  ```
  mount /dev/vg-Main/root /mnt
  mkdir /mnt/boot
  mount /dev/vg-Main/boot /mnt/boot
  mkdir /mnt/boot/efi
  mount /dev/sda1 /mnt/boot/efi
  swapon /dev/vg-Main/swap
  ```
* Bootstrap the initial system, answering yes to all questions.
  ```
  pacstrap -i /mnt base base-devel
  ```
* Package overview
  * base: Base system
  * base-devel: Base development, for AUR support later on
* Create the fstab files
  ```
  genfstab -p /mnt > /mnt/etc/fstab
  ```
* Change-root to the new filesystem
  ```
  arch-chroot /mnt /bin/bash
  ```
* Set up locales
  * Open /etc/locale.gen
  ```
  nano /etc/locale.gen
  ```
  * Uncomment 'en_US.UTF-8' 
  * Save & Quit
  * Generate locales
    ```
    locale-gen
    ```
  * Set up default locale:
    ```
    echo LANG=en_US.UTF-8 > /etc/locale.conf
    ```
* Set up timezone
  ```
  ln -s /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime
  hwclock --systohc --utc
  ```
* Set up hostname:
  ```
  echo Hydra > /etc/hostname
  ```
* Add hostname to both localhost names
  ```
  nano /etc/hosts
  ```
* Enable DHCP for main device (check interface with ip link)
  ```
  ip link
  systemctl enable dhcpcd@enp0s25.service
  systemctl enable netctl-ifplugd@enp0s25.service
  ```
* Set up init ram filesystem
  * Add lvm2 in HOOKS between block and filesystems
    ```
    nano /etc/mkinitcpio.conf
    ```
  * Create init ram fs
    ```
    mkinitcpio -p linux
    ```
* Set up GRUB
  * Install needed packages
    ```
    pacman -S grub efibootmgr
    ```
  * Add 'lvm' to GRUB_PRELOAD_MODULES
    ```
    nano /etc/default/grub
    ```
  * Install GRUB to EFI partition directory
    ```
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub --recheck
    ```
  * Create GRUB config
    ```
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
  * Set root password
    ```
    passwd
    ```
  * Exit chroot, and reboot:
    ```
    exit
    shutdown -r now
    ```

## Setup up graphical desktop and Google Chromium (for basic troubleshooting)
* Create a regular user for the desktop
  ```
  useradd -m -G wheel -s /bin/bash archie
  passwd archie
  ```
* Allow new user to use sudo
  * Open sudoers file
    ```
    nano /etc/sudoers
    ```
  * uncomment the following line:
    ```
    %wheel ALL=(ALL) ALL
    ```
* Install packages:
  ```
  pacman -S xorg-server xf86-video-intel lxde gvfs gvfs-smb chromium 
  ```
* Package descriptions:
  * xorg-server: X11
  * xf86-video-intel: Intel Integrated Graphics video driver
  * lxde: Lightweight X11 Desktop Environment
    (Answer 'all' when asked which packages from lxde to install, answer mesagl when asked for the GL provider)
  * gvfs gvfs-smb: Gnome virtual filesystem & Samba support (to be able to browse samba drives from the file manager)
  * chromium: Google Chromium browser (Answer ttf-freefont for the truetype font provider)
* Have LXDM start at boot
  ```
  systemctl enable lxdm
  ```
* Reboot
  ```
  shutdown -r now
  ```
