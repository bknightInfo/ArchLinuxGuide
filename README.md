# Arch installation on a BTRFS root filesystem with timeshift
Instructions to perform an installation of Arch Linux using BTRFS filesystem in /

## Create Arch ISO or Use Image

Download ArchISO from <https://archlinux.org/download/> and put on a USB drive with [Etcher](https://www.balena.io/etcher/), [Ventoy](https://www.ventoy.net/en/index.html), or [Rufus](https://rufus.ie/en/)

### No Wifi

Need Wifi go through these 3 steps:

#1: Run `iwctl`

#2: Run `station wlan0 get-networks`

#3: Find your network, and run `station wlan0 connect [network name]`, enter your password and run `exit`. 

You can test if you have internet connection by running `pacman -Syy`.

## Set Time

`timedatectl set-ntp true`

## Update mirror list 

`reflector --country Australia --protocol https --latest 10 --Sort Rate --save /etc/pacman.d/mirrorlist --verbose`

## list disk devices

`lsblk`

## Partitioning

`cfdisk -z /dev/[device]` e.g. cfdisk -z /dev/nvme0n1

[Arch Wiki] https://man.archlinux.org/man/cfdisk.8.en

Ideal setup

| Mount point | Partition | Partition type | FS Type     | Bootable flag | Size   |
|-------------|-----------|---------------------|-----------|----|--------|
| /boot/efi       | /dev/[Partition]1 | EFI System Partition| FAT32 | Yes           | 512 MiB|
| [SWAP]      | /dev/[Partition]2 | Linux swap | SWAP          | No            | 17 GiB |
| /           | /dev/[Partition]3 | Linux | BTRFS       | No            | remaining GB|

I will be using /dev/nvme0n1 moving forward


## Format Partitions

#1: `lsblk`

#2: `mkfs.fat -F32 /dev/nvme0n1p1`

#3: `mkswap /dev/nvme0n1p2`

#4: `swapon /dev/nvme0n1p2`

#5: `mkfs.btrs /dev/nvme0n1p3`


## Create BTRFS subvolumes

#1: `mount /dev/nvme0n1p3 /mnt`

#2: `cd /mnt`

#3: `btrfs subvolume create @`

#4: `btrfs subvolume create @home`

#5: `btrfs subvolume create @var`

#6: `btrfs subvolume create @data`

#7: `btrfs subvolume create @srv`

#8: `cd`

#9: `umount /mnt`


## Mount BTRFS subvolumes and EFI

#1: `mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@ /dev/nvme0n1p3 /mnt`

#2: `mkdir -p /mnt/{boot/efi,home,var,data,src}`

#3: `mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@home /dev/nvme0n1p3 /mnt/home`

#4: `mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@home /dev/nvme0n1p3 /mnt/var`

#5: `mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@data /dev/nvme0n1p3 /mnt/data`

#6: `mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@src /dev/nvme0n1p3 /mnt/srv`

#7: `mount /dev/nvme0n1p1 /mnt/boot/efi`

## Review partitions

#1: `lsblk`

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

#1: Run `gpacman -Syy`

#2: Run `gpacman -S acpi acpi_call acpid alsa-plugins alsa-utils avahi base-devel bluez bluez-utils cups dnsutils dosfstools efibootmgr grub gvfs gvfs-smb inetutils linux-headers mtools networkmanager network-manager-applet neofetch nfs-utils ntfs-3g openssh os-prober pipewire pipewire-alsa pipewire-jack pipewire-pulse reflector rsync system-config-printer tlp wpa_supplicant xdg-user-dirs xdg-utils`

## Install bootloader
#1: Run `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux`

#2: Run `sed -i 's/#GRUB_DISABLE_OS_PROBER=false/GRUB_DISABLE_OS_PROBER=false/g' /etc/default/grub`

#3: Run `grub-mkconfig -o /boot/grub/grub.cfg`

## Add new user

#1: Run `useradd -m -G wheel yourUsername`

#2: Run `passwd yourUsername`

#3: Run `uncomment "%wheel ALL=(ALL) ALL" in the following line`

#4: Run `export EDITOR=nano visudo`


## Update mirror list 

#1: Run `reflector --country Australia --protocol https --latest 10 --Sort Rate --save /etc/pacman.d/mirrorlist --verbose`

## start services

#1: Run `sudo systemctl enable acpid`

#2: Run `sudo systemctl enable avahi-daemon`

#3: Run `sudo systemctl enable bluetooth`

#4: Run `sudo systemctl enable cups`

#5: Run `sudo systemctl enable fstrim.timer`

#6: Run `sudo systemctl enable NetworkManager`

#7: Run `sudo systemctl enable reflector.timer`

#8: Run `sudo systemctl enable sshd`

#9: Run `sudo systemctl enable tlp`


## Reboot

#1: Run `gexit`

#1: Run `gumount -a`

#1: Run `greboot`

## Post install 

login as yourUsername

## Install AUR helper

#1: Run `git clone https://aur.archlinux.org/paru-bin`

#2: Run `cd paru-bin`

#3: Run `makepkg -si`

#4: Run `cd`

#5: Run `m -rf paru-bin`

## Install desktop (XCFE)

#1: Run `sudo pacman -S xfce4 xfce4-goodies lightdm vlc xorg amarok p7zip p7zip-plugins unrar tar`

## Codecs and Plugins
You will have to install codecs to play audio and video files.

#1: Run `sudo pacman -S a52dec faac faad2 flac jasper lame libdca libdv libmad libmpeg2 libtheora libvorbis libxv wavpack x264 xvidcore gstreamer0.10-plugins`

## AUR install packages

#1: Run `sparu -S zramd timeshift timeshift-autosnap brave-bin ttf-ms-fonts noto-fonts ttf-roboto ttf-inconsolata pamac-aur`

## Start remaining services

#1: Run `ssudo systemctl enable zramd`

#2: Run `ssudo systemctl enable lightdm`

## Troubleshooting

[Official Arch Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide) for more information.
