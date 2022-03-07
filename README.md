# Arch installation on a BTRFS root filesystem with timeshift
Instructions to perform an installation of Arch Linux using BTRFS filesystem in /

[TOC]

## Create Arch ISO or Use Image

Download ArchISO from <https://archlinux.org/download/> and put on a USB drive with [Etcher](https://www.balena.io/etcher/), [Ventoy](https://www.ventoy.net/en/index.html), or [Rufus](https://rufus.ie/en/)

### No Wifi

Need Wifi go through these 3 steps:

#1: Run `iwctl`

#2: Run `station wlan0 get-networks`

#3: Find your network, and run `station wlan0 connect [network name]`, enter your password and run `exit`. 

You can test if you have internet connection by running `pacman -Syy`.

## Set Time

timedatectl set-ntp true

## Update mirror list 

reflector --country Australia --protocol https --latest 10 --Sort Rate --save /etc/pacman.d/mirrorlist --verbose

## list disk devices

lsblk

## Partitioning

cfdisk -z /dev/[device] e.g. cfdisk -z /dev/nvme0n1

[Arch Wiki] https://man.archlinux.org/man/cfdisk.8.en

Ideal setup

| Mount point | Partition | Partition type | FS Type     | Bootable flag | Size   |
|-------------|-----------|---------------------|-----------|----|--------|
| /boot/efi       | /dev/sda1 | EFI System Partition| FAT32 | Yes           | 512 MiB|
| [SWAP]      | /dev/sda2 | Linux swap | SWAP          | No            | 17 GiB |
| /           | /dev/sda3 | Linux | BTRFS       | No            | remaining GB|

I will be using /dev/nvme0n1 moving forward


## Format Partitions

lsblk

mkfs.fat -F32 /dev/nvme0n1p1

mkswap /dev/nvme0n1p2

swapon /dev/nvme0n1p2

mkfs.btrs /dev/nvme0n1p3


## Create BTRFS subvolumes

mount /dev/nvme0n1p3 /mnt

cd /mnt

btrfs subvolume create @

btrfs subvolume create @home

btrfs subvolume create @var

btrfs subvolume create @data

btrfs subvolume create @src

cd

umount /mnt

## Mount BTRFS subvolumes and EFI

mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@ /dev/nvme0n1p3 /mnt

mkdir -p /mnt/{boot/efi,home,var,data,src}

mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@home /dev/nvme0n1p3 /mnt/home

mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@home /dev/nvme0n1p3 /mnt/var

mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@data /dev/nvme0n1p3 /mnt/data

mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@src /dev/nvme0n1p3 /mnt/src

mount /dev/nvme0n1p1 /mnt/boot/efi

## Review partitions

lsblk

Partitions should look like this

| Partition     | Mount point |
[---------------|-------------|
|/dev/nvme0n1p1 | /boot/efi   |
|/dev/nvme0n1p2 | [SWAP]      |
|/dev/nvme0n1p1 | /home       |
                | /var        |
                | /src        |
                | /data       |
                | /           |

## Install base files

pacstrap /mnt base btrfs-progs git amd-ucode linux linux-firmware nano vim 

With Intel systems replace amd-ucode with intel-ucode

## Generate file system 

genfstab -U /mnt >> /mnt/etc/fstab 

## Check fstab file

cat /mnt/etc/fstab

# Chroot

arch-chroot /mnt 

## Set Timezone

ln -sf /usr/share/zoneinfo/Australia/Melbourne /etc/localtime

## Set hardware clock

hwclock --systohc

## Set localization

nano /etc/locale.gen 

locale-gen

echo "LANG=en_AU.UTF-8" >> /etc/locale.conf 

echo "YourHost" >>  /etc/hostname 

echo "127.0.0.1   localhost" >> /etc/hosts
echo "1::1         localhost" >> /etc/hosts
echo "1127.0.0.1   YourHost.localdomain YourHost" >> /etc/hosts

## Set root password

passwd

## Add btrfs module to mkinitcpio

In modules add btrfs then save

nano /etc/mkinitcpio.conf

mkinitcpio -p linux


## Install the rest of base install

pacman -Syy

pacman -S acpi acpi_call acpid alsa-plugins alsa-utils avahi base-devel bluez bluez-utils cups dnsutils dosfstools efibootmgr grub gvfs gvfs-smb inetutils linux-headers mtools networkmanager network-manager-applet neofetch nfs-utils ntfs-3g openssh os-prober pipewire pipewire-alsa pipewire-jack pipewire-pulse reflector rsync system-config-printer tlp wpa_supplicant xdg-user-dirs xdg-utils 

## Install bootloader
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux

sed -i 's/#GRUB_DISABLE_OS_PROBER=false/GRUB_DISABLE_OS_PROBER=false/g' /etc/default/grub

grub-mkconfig -o /boot/grub/grub.cfg  

## Add new user

useradd -m -G wheel yourUsername

passwd yourUsername 

uncomment "%wheel ALL=(ALL) ALL" in the following line

export EDITOR=nano visudo


## Update mirror list 

reflector --country Australia --protocol https --latest 10 --Sort Rate --save /etc/pacman.d/mirrorlist --verbose

## start services

sudo systemctl enable acpid

sudo systemctl enable avahi-daemon

sudo systemctl enable bluetooth

sudo systemctl enable cups

sudo systemctl enable fstrim.timer

sudo systemctl enable NetworkManager

sudo systemctl enable reflector.timer

sudo systemctl enable sshd

sudo systemctl enable tlp


## Reboot

exit

umount -a

reboot

## Post install 

login as yourUsername

## Install AUR helper

git clone https://aur.archlinux.org/paru-bin

cd paru-bin

makepkg -si

cd

rm -rf paru-bin 

## Install desktop (XCFE)

sudo pacman -S xfce4 xfce4-goodies libreoffice-fresh lightdm firefox vlc xorg

## AUR install packages

paru -S zramd timeshift timeshift-autosnap brave-bin ttf-ms-fonts noto-fonts ttf-roboto ttf-inconsolata

## start remaining services

sudo systemctl enable zramd

sudo systemctl enable lightdm

## Troubleshooting

[Official Arch Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide) for more information.
