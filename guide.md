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
  * Open /etc/default/grub
    ```
    nano /etc/default/grub
    ```
  * Add 'lvm' to GRUB_PRELOAD_MODULES
  * Uncomment the line: (this helps out with booting LVM snapshots in the future)
    ```
    GRUB_DISABLE_LINUX_UUID=true
    ```
  * Save & Quit
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

## LVM Snapshots
Now is a good time to show how to experiment. If you want to install stuff, change stuff, or otherwise break stuff, you can make an LVM snapshot. You can do so for root and boot, but unless you do kernel changes, root is probably enough.

Create a snapshot using:
```
lvcreate -s --name root-snapshot --size 1GB /dev/vg-Main/root
```
This will create a snapshot with a maximum of 1GB of changes (if you change more than that, the snapshot will break and disappear).

If for some reason you break something, you can boot using the ArchLinux Live CD, and type:
```
lvconvert --merge /dev/vg-Main/root-snapshot
```
(Note that this also removes the snapshot!)

If you're happy with the setup and would like to remove the snapshot, type:
```
lvremove /dev/vg-Main/root-snapshot
```

The same commands can also be used to snapshot boot.

If you have space enough, I recommend making the snapshot size equal to the original volumes size, just to make sure you never run out of space in the snapshot.

## Yaourt

To be able to get packages from the ArchLinux User Repository (AUR), it's easiest to install Yaourt. First, as root, install wget:
```
pacman -S wget
```
Then, as your normal user, run the following commands:
```
mkdir yaourt
cd yaourt
wget https://aur.archlinux.org/packages/pa/package-query/package-query.tar.gz
wget https://aur.archlinux.org/packages/ya/yaourt/yaourt.tar.gz
tar xvf package-query.tar.gz
tar xvf yaourt.tar.gz
cd package-query
makepkg -si
cd ../yaourt
makepkg -si
```
## Getting the VGA devices ready

We should make sure that no drivers are loaded for the devices we'd like to pass through, so we'll force the pci-stub drivers to be loaded for them.
* Find the PCI-ids (in the form of XXXX:XXXX) and PCI locations (XX:XX.X) for the devices you'd like to pass through.
  ```
  lspci -nn | grep NVIDIA
  ```
* My output looks like:
  ```
  01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GT200b [GeForce GTX 275] [10de:05e6] (rev a1)
  02:00.0 VGA compatible controller [0300]: NVIDIA Corporation G96 [GeForce 9500 GT] [10de:0640] (rev a1)
  ```
* So the PCI-ids for my devices are 10de:05e6 and 10de:0640, and the locations are 01:00.0 and 02:00.0.
* Add the pci-ids to your GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub: pci-stub.ids=10de:05e6,10de:0640
* Create the new grub config:
  ```
  grub-mkconfig -o /boot/grub/grub.cfg
  ```
* Make sure pci-stub is added to the initramfs:
  * Edit /etc/mkinitcpio.conf and add pci-stub to the MODULES option
  * Recreate the initramfs
    ```
    mkinitcpio -p linux
    ```
* Reboot
  ```
  shutdown -r now
  ```
* Check which devices are claimed by pci-stub
  ```
  dmesg | grep pci-stub
  ```
* You should make sure you can find lines like this for all your devices:
  ```
  [    0.605687] pci-stub: add 10DE:0640 sub=FFFFFFFF:FFFFFFFF cls=00000000/00000000
  [    0.605690] pci-stub 0000:02:00.0: claimed by stub
  ```
* Optionally add 'nouveau' and/or 'radeon' to the modules blacklist (prevent them from ever getting loaded)
  ```
  echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf 
  echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf 
  ```

## QEMU & Drivers
Before we go any further, let's get VGA passthrough working.
First install QEMU, you can either use the version in the ArchLinux repository:
```
pacman -S qemu
```
Or you can get the absolute latest version from AUR: (this is what I did).
```
yaourt -Q qemu-git
```
## Kernel patches for Intel Integrated Graphics
https://aur.archlinux.org/packages/linux-vfio/

## Sound