Arch on Air (please don't follow yet, this is incomplete right now)
---------

Instructions for installing Arch Linux side-by-side with OS X on a Macbook Air 2013/2014.

This is a fork of [this](https://github.com/pandeiro/arch-on-air) to include encryption using dm-crypt.

Most of this information was taken from these two sources:
- [https://bbs.archlinux.org/viewtopic.php?id%3D165899](ArchLinux Forums: Macbook Air 2013)
- [http://panks.me/blog/2013/06/arch-linux-installation-with-os-x-on-macbook-air-dual-boot/](ArchLinux Installation With OS X on Macbook Air (Dual Boot))
- [https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encrypting_devices_with_cryptsetup](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encrypting_devices_with_cryptsetup)

## Procedure
1. Make bootable USB media with Arch ISO image ([https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media](wiki))
2. Hold the `<alt/option>` key and boot into USB
3. Create partitions
The following example assumes Arch will sit on a single partition;
adjust according to preference.
You might want a swap partition, but in my opinion it is not needed if you have 8GB of RAM but a rather small disk (128GB).

```
cgdisk /dev/sda
```

Partitions:
* [128MB] Apple HFS+ "Boot Loader" (will get formatted to HFS+ later)
* [256MB] Linux filesystem "boot"
* [Rest of space] Linux filesystem "root"
That's exactly three new partitions. Don't forget /boot. It would work without it, but not with disk encryption (happened to me)

### Very important: device names

These can vary, so be careful and double-check everything involving the partitions (`lsblk` is nice).
* `/dev/sda4` is assumed to be the apple bootloader
* `/dev/sda5` is assumed to be the /boot partition of the linux system
* `/dev/sda6` is assumed to be the root file system.

## 4. Format and mount partition
### Encryption setup
See [the wiki page](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Simple_partition_layout_with_LUKS) for more information.
#### Optional: Wipe data on disk (don't choose the wrong one!)
##### Some arguments for it can be found [here](https://wiki.archlinux.org/index.php/Disk_encryption#Preparing_the_disk).
See [this wiki page](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#Secure_erasure_of_the_hard_disk_drive).
```sh
## Create a new encrypted container
cryptsetup open --type plain /dev/sda5 container
## Type in any password
## check that it exists
lsblk
ls /dev/mapper # (should have one "container")
## write zeroes over it. These zeroes will get encrypted and will not be
## distuinguishable from random data (/dev/random).
dd if=/dev/zero of=/dev/mapper/container
```

```
cryptsetup /dev/sda6 
```


```
mkfs.ext4 /dev/sda5 # /boot
mkfs.ext4 /dev/sda6 # /
mount /dev/sda6 /mnt
mkdir /mnt/boot
mount /dev/sda5 /mnt/boot
```


## 5. Installation
This requires an internet connection. Options:
- Tethered phone via USB (easiest IMO)
- Wired (with some Apple proprietary ethernet thing ($$$?))
- Wireless (requires b43 wireless firmware ([[https://aur.archlinux.org/packages/b43-firmware/][AUR]]))
```
begin_src sh
pacstrap /mnt base base-devel
genfstab -U -p /mnt >> /mnt/etc/fstab
```


## 6. Optimize fstab for SSD
```
nano /mnt/etc/fstab
```
```
/dev/sda6 /     ext4 defaults,noatime,discard,data=writeback 0 1
/dev/sda5 /boot ext4 defaults,relatime,stripe=4              0 2
```
https://wiki.archlinux.org/index.php/Solid_State_Drives
It's about 'discard'. TODO more info


## 7. Configure system
```
arch-chroot /mnt /bin/bash
passwd
echo myhostname > /etc/hostname
## https://wiki.archlinux.org/index.php/beginners%27_guide#Time_zone
ls /usr/share/zoneinfo # find your region
ln -s /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
# ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc --utc
useradd -m -g users -G wheel -s /bin/bash myusername
passwd myusername
pacman -S sudo
```


## 8. Grant sudo
```
echo "myusername ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```


## 9. Set up locale
```
nano /etc/locale.gen
```
```
locale-gen
echo LANG=en_US.UTF8 > /etc/locale.conf
export LANG=en_US.UTF-8
```


## 10. Set up mkinitcpio hooks and run
Insert "keyboard" after "autodetect" if it's not already there.
```
nano /etc/mkinitcpio.conf
```
Then run it:
```
mkinitcpio -p linux
```


## 11. Set up GRUB/EFI
To boot up the computer we will continue to use Apple's EFI
bootloader, so we need GRUB-EFI:
```
pacman -S grub-efi-x86_64
```
### Configuring GRUB
```
nano /etc/default/grub
```
Aside from setting the quiet and rootflags kernel parameters,
a special parameter must be set to avoid system (CPU/IO)
hangs related to ATA, as per [[https://bbs.archlinux.org/viewtopic.php?pid%3D1295212#p1295212][this thread]]:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback libata.force=1:noncq"
```
Additionally, the grub template is broken and requires this adjustment:
```
# fix broken grub.cfg gen
GRUB_DISABLE_SUBMENU=y
```
```sh
grub-mkconfig -o boot/grub/grub.cfg
grub-mkstandalone -o boot.efi -d usr/lib/grub/x86_64-efi -O x86_64-efi --compress=xz boot/grub/grub.cfg
## might be useful (if you already have one USB stick and one with WiFi):
nc -l 8181 > boot.efi

nc 192.168.178.168 8181 < boot.efi
```
Copy boot.efi (generated in the command above) to a USB stick for use later in OS X.



## 12. Setup boot in OS X
Exit everything and reboot into OS X (by holding alt/option) and
then choosing it.
```sh
exit # exit chroot
reboot
```


## 13. Launch Disk Utility in OS X
Format ("Erase") /dev/sda4 using Mac journaled filesystem.
Yes, really "Erase".


## 14. Create boot file structure
This procedure allows the Apple bootloader to see our Arch
Linux system and present it as the default boot option.
```sh
cd /Volumes/disk0s4
mkdir System mach_kernel
cd System
mkdir Library
cd Library
mkdir CoreServices
cd CoreServices
touch SystemVersion.plist
```
```sh
nano SystemVersion.plist
```
```xml
<xml version="1.0" encoding="utf-8"?>
<plist version="1.0">
<dict>
    <key>ProductBuildVersion</key>
    <string></string>
    <key>ProductName</key>
    <string>Linux</string>
    <key>ProductVersion</key>
    <string>Arch Linux</string>
</dict>
</plist>
```
Copy boot.efi from your USB stick to this CoreServices directory.
The tree should look like this:
```
|___mach_kernel
|___System
       |
       |___Library
              |
              |___CoreServices
                      |
                      |___SystemVersion.plist
                      |___boot.efi
```


## 15. Make Boot Loader partition bootable
```sh
sudo bless --device /dev/disk0s4 --setBoot
```
Voila, Arch Linux is installed.

Reboot the computer and hold the alt/option key to
select which operating system to boot.


## 16. Get wireless working in Arch
### Get broadcom drivers
###  Download and install [[https://aur.archlinux.org/packages/broadcom-wl/][broadcom wl from AUR]]
(Make sure that b43 and ssb modules are not present in the output
from `lsmod`)
```sh
modprobe wl
```
### Alternatively, install [[https://aur.archlinux.org/packages/broadcom-wl-dkms/][broadcom-wl-dkms]] instead
...so that kernel updates don't leave you without wifi. DKMS
is a service that recompiles external modules after every kernel
upgrade.
```sh
sudo pacman -S dkms
sudo systemctl enable dkms.service
```
### Select network
```sh
sudo pacman -S dialog
sudo wifi-menu -o
```


## 17. Tilde key
The tilde key does not work on the keyboard out of the box. There
are several solutions listed [[https://wiki.archlinux.org/index.php/Apple_Keyboard][here]] but this one worked for me:
```sh
sudo nano /etc/modprobe.d/hid_apple.conf
```
```sh
options hid_apple iso_layout=0
```


## 18. Insert and <F1..12> keys
The <insert> key can be reproduced with fn+<Enter>. So to paste in an xterm
window for instance, use S-fn-<Enter>.

F1-F12 require fn+<F1>, etc.