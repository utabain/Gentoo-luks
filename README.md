# Arch Linux Full-Disk Encryption Installation Guide
### Set the console keyboard layout
The default console keymap is US. Available layouts can be listed with:
```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```
To set the keyboard layout, pass a corresponding file name to loadkeys, omitting path and file extension. For example, to set a German keyboard layout:
```
# loadkeys de-latin1
```
## Connect to the internet
> Ethernet should be automatic in the arch iso, if any errors occour be sure to consult the archwiki or community fourm.

### Connect to wifi (optional):
```
# iwctl
```
The interactive prompt is displayed with the prefix of `[iwd]#`.

Firstly, if you do not know your wireless device name, list all Wi-Fi devices:
```
[iwd]# device list
```
Then, to initiate a scan for networks (note that this command will not output anything):
```
[iwd]# station device scan
```
You can then list all available networks:
```
[iwd]# station device get-networks
```
Finally, to connect to a network:
```
[iwd]# station device connect SSID
```
If a passphrase is required, you will be prompted to enter it. Alternatively, you can supply it as a command line argument:
```
# iwctl --passphrase passphrase station device connect SSID
```
The connection may now be verified with ping,

(Use the `Ctrl Z` shortcut to kill the command once the connection has been confirmed):
```
# ping archlinux.org
```
### Update the system clock
To ensure that the system clock is accurate, please run this command:
```
# timedatectl set-ntp true
```
## Partition the disk
When recognized by the live system, disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use the
command bellow:
```
# fdisk -l
```
Zap the disk,

(**WARNING** This will irreversibly delete all data on your disk, please backup any important data):
```
sgdisk --zap-all /dev/nvme0n1
```
Create the partitions using the `sgdisk` command:
Number |           Size          | Code |    Type    |
-------|-------------------------|------|------------|
   1   | 550.0 MiB               | EF00 | EFI System |
   2   | Remainder of the device | 8309 | Linux LUKS |
```
sgdisk --clear \
       --new=1:0:+550MiB --typecode=1:ef00 \
       --new=3:0:0       --typecode=2:8309 /dev/nvme0n1
```
Create the LUKS2 encrypted container on the Linux LUKS partition:
```
cryptsetup luksFormat -h sha512 -i 5000 /dev/nvme0n1p2
```
Open the newly created container:
```
cryptsetup open /dev/nvme0n1p2 cryptlvm
```
## Preparing the logical volumes
Create a physical volume on top of the opened LUKS container:
```
# pvcreate /dev/mapper/cryptlvm
```
Create the volume group and add physical volume to it:
```
# vgcreate vg /dev/mapper/cryptlvm
```
Create logical volumes on the volume group for swap and root,
(Swap size is a matter of personal preference, 8GB is just a placeholder):
```
# lvcreate -L 8G vg -n swap
# lvcreate -l 100%FREE vg -n root
```
Create a fileystem for ext4 to `/dev/vg/root` using the mkfs command:
```
# mkfs.ext4 /dev/vg/root
```
Create a filesystem for `/dev/vg/swap` using the mkswap command:
```
# mkswap /dev/vg/swap
```
Create a filesystem for the EFI System partition:
```
# mkfs.fat -F32 /dev/nvme0n1p1
```
Disabling journaling:
```
# tune2fs -O "^has_journal" /dev/vg/root
```
Mount `/dev/nvme0n1p2` to `/mnt/boot`
```
# mount /dev/nvme0n1p2 /mnt/boot
```
Mount `/dev/vg/root` to `/mnt`:
```
# mount /dev/vg/root /mnt
```
Activate the swap file:
```
# swapon /dev/vg/swap
```
## Installation
Install necessary packages:
```
# pacstrap /mnt base linux linux-firmware mkinitcpio lvm2 vim dhcpcd wpa_supplicant network_manager sbsigntools dracut efibootmgr git
```
## Configure the system (Pre-chroot)

Generate an fstab file:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
(optional) Change relatime option to noatime
| /mnt/etc/fstab                                                  |
| --------------------------------------------------------------- |
| # /dev/mapper/cryptlvm                                          |
| UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx / ext4 rw,noatime 0 1 |

## Change root into the new system
Use the `chroot` command to acomplish the above:
```
# arch-chroot /mnt
```
At this point you should have the following partitions and logical volumes:

('100%FREE' is just a place holder for the remainder of your drive aswell as 'chosen ammount' being a place holder for the ammount of swap space you dedicated to the swap partition)

`lsblk`
NAME           | MAJ:MIN | RM  |  SIZE          | RO  | TYPE  | MOUNTPOINT |
---------------|---------|-----|----------------|-----|-------|------------|
nvme0n1        |  259:0  |  0  | 100%FREE       |  0  | disk  |            |
├─nvme0n1p1    |  259:3  |  0  | 512M           |  0  | part  | /efi       |
├─nvme0n1p2    |  259:4  |  0  | 100%FREE       |  0  | part  |            |
..└─cryptlvm   |  254:0  |  0  | 100%FREE       |  0  | crypt |            |
....├─vg-swap  |  254:1  |  0  | chosen ammount |  0  | lvm   | [SWAP]     |
....├─vg-root  |  254:2  |  0  | 100%FREE       |  0  | lvm   | /          |

## Configuring the system (Post-chroot)
### Timezone
To configure the timezone correctly replace `Europe/London` with your respective timezone found in `/usr/share/zoneinfo`
```
ln -sf /usr/share/zoneinfo/Europe/:ondon /etc/localtime
```
Run `hwclock` to generate `/etc/adjtime`:

(Assumes hardware clock is set to UTC)
```
# hwclock --systohc
```
### Localization
Uncomment `en_GB.UTF-8 UTF-8` in `/etc/locale.gen` and generate the locale,

('en_GB.UTF-8 UTF-8' is just a placeholder, replace with your respective locale):
```
# locale-gen
```
The default console keymap is US. Available layouts can be listed with:
```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```
If you set the console keyboard layout, make the changes persistent in `vconsole.conf`,

(This example uses the German console keymap, please change this to your preffered console keymap):
```
echo KEYMAP=de-latin1 > /etc/vconsole.conf
```
### Network configuration
Please create a host name file, keep in mind that the hostname is a unique name for identifying your machine on a network,

('myhostname' is just a placeholder, rename it to anything that you'd like):
```
echo myhostname > /etc/hostname
```
## Initramfs
Add the `systemd`, `keyboard`, `sd-vconsole`, `sd-encrypt` and `lvm2` hooks to `/etc/mkinitcpio.conf`,

(ordering matters):
```
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt lvm2 filesystems)
```
Recreate the initramfs image:
```
mkinitcpio -p linux
```
## Install microcode
> Processor manufacturers release stability and security updates to the processor microcode. These updates provide bug fixes that can be critical to the stability of your system. Without them, you may experience spurious crashes or unexpected system halts that can be difficult to track down.
> 
> All users with an AMD or Intel CPU should install the microcode updates to ensure system stability.
```
pacman -S intel-ucode
```
## Install and configure systemd-boot
Use `bootctl` to install systemd-boot into the EFI system partition:
```
# bootctl --path=/boot install
```
Find the UUID of your root partition (not the encrypted volume within):
```
blkid | grep /dev/vg/root | awk '{print $2}'
```
Create a loader entry `/boot/loader/entries/arch.conf` of your Arch Linux installation,

('Your_UUID' is just a place holder, replace with the results from the command above):
```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img # only if you are using an Intel CPU
initrd /initramfs-linux.img
options rd.luks.name=Your_UUID=root rd.luks.options=fido2-device=auto root=/dev/mapper/vg-root
```
Edit `/boot/loader/loader.conf` to look like this:
```
default arch
timeout 4
console-mode max
editor no
```
Restrict access to the `/boot` directory:
```
chmod 700 /boot
```
## Disk decryption with FIDO2/YubiKey
Enroll the key:
```
systemd-cryptenroll --fido2-device=auto /dev/mapper/vg-root
```
Edit `/etc/crypttab.initramfs` (may be nonexistent or empty) and add the following lines:
```
# <name>    <device>    <password>  <options>
root	/dev/mapper/vg-root	-	fido2-device=auto
```
## Exiting chroot
Set the root password:
```
passwd
```
Exit the chroot enviorment:
```
exit
```
Reboot the system:
```
reboot
```
## Post-install
> Your system should now be fully installed, bootable, and fully encrypted.
> 
> For the standard Arch Linux post-installation steps, [RTFM](https://wiki.archlinux.org/index.php/General_recommendations).
### Connect to wifi (optional)
Enable Your Wi-Fi Device:
```
nmcli radio wifi on
```
Identify a Wi-Fi Access Point:
```
nmcli dev wifi list
```
Connect to Wi-Fi,

('network-ssid' is just a placeholder. Replace this with the ssid of your network, avalible networks can be shown with the command above):
```
sudo nmcli dev wifi connect network-ssid
```
If you have WEP or WPA security on your WI-Fi, you can specify the network password in the command as well,

('network-ssid' and 'network-password' are both placeholders, replace with the correct values):
```
sudo nmcli dev wifi connect network-ssid password "network-password"
```
### Preparation: booting from a unified kernel image
A Unified Kernel Image is a compilation containing the following:

> • UEFI bootloader executable
> 
> • Linux kernel
> 
> • initramfs
> 
> • Kernel command-line arguments

Edit `/etc/mkinitcpio.d/linux.preset` and add the following lines:
```
ALL_microcode=(/boot/*-ucode.img)
default_efi_image="/boot/EFI/Linux/linux.efi"
default_options=""
fallback_efi_image="/boot/EFI/Linux/fallback.efi"
```
Edit the line starting with fallback_options in `/etc/mkinitcpio.d/linux.preset` to contain:
```
fallback_options="-S autodetect --splash /usr/share/systemd/bootctl/splash-arch.bmp"
```
Remove any references to initrd/initramfs:
```
cat /proc/cmdline > /etc/kernel/cmdline
```
Recreate the initramfs image:
```
mkinitcpio -P linux
```
Reboot and make sure that you have two new entries in your systemd-boot menu:

one for Arch Linux, and one for Arch Linux fallback.
### Enrolling your key into secure boot
> Reboot into your UEFI interface and enable secure boot. Set the secure boot mode setting to 
>
> “Setup mode,” which allows enrolling new keys. Then boot back into Arch.

Install `sbctl`:
```
pacman -S sbctl
```
Create a private key:
```
sbctl create-keys
```
Enroll the newly created keys:
```
sbctl enroll-keys
```
Sign each of the EFI files that may appear somewhere in the boot chain:
```
sbctl sign -s /boot/EFI/Linux/linux.efi
sbctl sign -s /boot/EFI/Linux/fallback.efi
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/EFI/Boot/bootx64.efi
```
Reboot into the UEFI interface and ensure that Secure Boot is still enabled. Verify that the Secure Boot mode setting has changed to "User mode".

Test booting into Arch and Arch fallback. All should succeed without issues.

## Sources
> https://wiki.archlinux.org/title/installation_guide
>
> https://wiki.archlinux.org/title/Iwd#iwctl
>
> https://wiki.archlinux.org/title/GPT_fdisk
>
> https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268
>
> https://wiki.archlinux.org/title/ext4
>
> https://wiki.archlinux.org/title/systemd-boot
>
> https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html
>
> https://man.archlinux.org/man/nmcli.1.en
>
> https://saligrama.io/blog/post/upgrading-personal-security-evil-maid/#disk-decryption-with-fido2yubikey

