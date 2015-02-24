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
## Getting the PCI devices ready for passthrough

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
* Create a file named ```/usr/bin/vfio-bind``` with the following contents:
  ```
  #!/bin/bash

  modprobe vfio-pci

  for dev in "$@"; do
          vendor=$(cat /sys/bus/pci/devices/$dev/vendor)
          device=$(cat /sys/bus/pci/devices/$dev/device)
          if [ -e /sys/bus/pci/devices/$dev/driver ]; then
                  echo $dev > /sys/bus/pci/devices/$dev/driver/unbind
          fi
          echo $vendor $device > /sys/bus/pci/drivers/vfio-pci/new_id
  done
  ```
* Make it executable:
  ```
  chmod 755 /usr/bin/vfio-bind
  ```
* Try to bind a device: (prepend the device with 0000:)
  ```
  vfio-bind 0000:01:00.0
  ```
* Look in ```dmesg```. If vfio-pci failed, you might need to double-check VT-d support:
  * If it failed, there will be a message with vfio-pci, if it succeeded, it'll just say 'VFIO - User Level meta-driver version: 0.3'
  * Go into the BIOS/UEFI, and check it is enabled.
  * Add intel_iommu=on to GRUB_CMDLINE_LINUX_DEFAULT in ```/etc/default/grub```
  * Recreate the grub config file: ```grub-mkconfig -o /boot/grub/grub.cfg```
  * Reboot ```shutdown -r now```
  * Try again ``` vfio-bind 0000:01:00.0```
  * Check again ```dmesg```
  * If it still doesn't work, try and find other instructions for your motherboard before going on with this guide.
* Create a system.d service, ```/usr/lib/systemd/system/vfio-bind.service``` with the following contents:
  ```
  [Unit]
  Description=Binds devices to vfio-pci
  After=syslog.target
  
  [Service]
  EnvironmentFile=-/etc/vfio-pci.cfg
  Type=oneshot
  RemainAfterExit=yes
  ExecStart=-/usr/bin/vfio-bind $DEVICES
  
  [Install]
  WantedBy=multi-user.target
  ```
* Create a config file for the systemd service in  ```/etc/vfio-pci.cfg```:
   ```
   DEVICES="0000:01:00.0 0000:02:00.0"
   ```
* Start and enable the service:
  ```
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
We can check if QEMU is working with KVM using the following command line (as root):
```
qemu-system-x86_64 -enable-kvm -m 1024 -cpu host -smp 1,sockets=1,cores=1,threads=1
```

This should open a window with a fake BIOS starting up complaining that it can't boot from CD or boot disk. If not, check if VT-x is enabled in your BIOS, and troubleshoot linux KVM. This means that at least QEMU is working.

## Kernel patches for Intel Integrated Graphics
https://aur.archlinux.org/packages/linux-vfio/

## Sound