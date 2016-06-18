# debian-p50

Copied from an internal issue.

## Unbox, secure boot, bios

Finally got to opening this beauty.

The box was incredibly spartan. Two plastic shells held the machine in the center of the box and a smaller cardboard box held the (frankly enormous) power brick. Better hope that this battery really does go all day cause I sure don't want to carry my power cable around.

I plugged it in and turned it on to get the Windows install booted so I could create an external recovery drive as I do not keep any of the Windows partition stuff around.

Did that, then opened the bios menu and made the following changes:

- Disabled secure boot (needed to boot my Linux install disk)
- Set Bios to UEFI Only, CSM Support: No
- Enabled virtualization extensions
- Bumped total graphics memory from 256 to 512MB
- Set the function keys to be function keys by default and not the "Special function" keys

Exiting the bios and saving changes. Time to try this Debian operating system some kids here are so crazy about.

## Installation media

Debian stable netinstaller image booted up just fine. I don't really know what I'm doing with debian so I went for the graphical expert install. Let's see how wrong that was.

Couldn't figure out how to get Debian's installer to use my preferred LVM on LUKS setup through it's labyrinth of installer menus. When I first started using Arch the fact that in order to install it I had to set up my own partition tables and then untar a base filesystem was such an unreliable install method and now I really wish that's what deb would let me do. Had to give up last night.

After playing around with Debian's installer settings. It doesn't look like it wants to give me power overwhelming to roll my own crypto parameters. It gives me some flags but I really want to twiddle all the knobs myself so that when I inevitably trash this install I know exactly how to access the encrypted volume from a recovery OS.

Can I use debootstrap to install a Debian system meant for actual hardware or is it for container OSes only?

Downloading an [LMDE]() live disk image to use as my installer environment. I'm choosing LMDE since Mint is what I use to recover stuff from the borked Windows installations of people in my life and LMDE is Debian sid based so should give me a good approximation of Debian quirks to watch out for as I bootstrap my own system.

> Yeah, you can deboostrap whatever you'd like; it won't normally install a kernel or a bootloader for you though, so you'd need to do those yourself afterwards.

Cool, the bootloader is one of the things I didn't want the installer to handle for me anyway since I'm just gonna efistub it.

well apparently LMDE's live image is a year old. We'll see how that goes with a brand new machine... Might just end up doing a Debian live image.

Built a debian live image, it didn't boot. Found this gem [in the docs](https://wiki.debian.org/UEFI#ARM64_platform:_UEFI.2C_U-Boot.2C_Fastboot.2C_etc.)

> ### UEFI support in live images
> 
> At this point, UEFI support exists only in Debian's installation images. The accompanying live images do not have support for UEFI boot, as the live-build software used to generate them still does not include it. Hopefully the debian-live developers will add this important feature soon. 

https://www.debian.org/releases/stable/amd64/apds03.html.en is probably the guide I want, I think I'm just gonna have to install it from Archlinux. lol

# Setting up partitions

```
# cfdisk /dev/nvmen0
```

Set up a 4GB efi partition `efiboot`

Set the rest up as generic Linux filesystem

```
# mkfs.fat -F32 -n efiboot /dev/nvme0n1p1
# cryptsetup luksFormat -c aes-xts-plain64 -s 512 -h sha512 -i 5000 --use-random -y /dev/nvme0n1p2
# cryptsetup open /dev/nvme0n1p2 navi
# pvcreate /dev/mapper/navi
# vgcreate navi /dev/mapper/navi
# lvcreate -n debian -L20G navi
# lvcreate -n home -L200G navi
# mkfs.ext4 /dev/mapper/navi-debian
# mkfs.ext4 /dev/mapper/navi-home
```

### Debootstrap

Set up wifi.

```
# wifi-menu
```

debootstrap is actually in Arch's community repo.

```
# pacman -Sy debootstrap
```

```
# mkdir /mnt/debinst
# mount /dev/mapper/navi-debian /mnt/debinst
# mkdir /mnt/debinst/{home,boot}
# mount /dev/mapper/navi-home /mnt/debinst/home
# mount /dev/disk/by-partlabel/efiboot /mnt/debinst/boot
# debootstrap --arch amd64 sid /mnt/debinst http://ftp.debian.org/debian

...
I: Base system installed successfully.
```

Omigosh a debian!

# System initialization

There's a handy-dandy archlinux utility called genfstab, I copied it to the chroot so I could use it

## Finalize mounting and such

```
# cp /usr/bin/genfstab /mnt/debinst/usr/bin/genfstab
# PATH=/bin:/usr/bin:/sbin:/usr/sbin LANG=C.UTF-8 chroot /mnt/debinst /bin/bash
# apt install makedev
# mount none /proc -t proc
# cd /dev
# MAKEDEV generic
# genfstab -L / >> /etc/fstab
# mount -t proc proc /proc
```

## Configure timezone data
```
# cat <<ADJTIME > /etc/adjtime
0.0 0 0.0
0
UTC
ADJTIME
# dpkg-reconfigure tz-data
```

## Networking

add `contrib` and `non-free` to `/etc/apt/sources.list`

```
# apt update
# apt install firmware-iwlwifi
````

## Locales, Console, and Keyboard

```
# apt install locales
# dpkg-reconfigure locales
# apt install console-setup xfont-terminus
# dpkg-reconfigure keyboard-configuration
```

## Kernel :corn: 

```
# apt install linux-image-amd64 linux-headers-amd64 firmware-linux-nonfree
```

## Boot loader

```
# mount -t sysfs sys /sys
# mount -t efivarfs efivarfs /sys/firmware/evi/efivars
```

I had to exit the chroot to install systemd-boot (nee gummiboot).

```
# bootctl --path=/mnt/debinst/boot install
```

```
# /boot/loader/entries/debian-sid.conf
title Debian Sid
linux vmlinuz-4.6.0-1-amd64
initrd initrd.img-4.6.0.1-amd64
options cryptdevice=PARTLABEL=navi:navi root=/dev/mapper/navi-debian
```

### Interlude

Debian what is this, I needed to `apt install man-db` in order to get /usr/bin/man

## Initramfs boot problems

Rebooted and no joy. Debian's magic to detect when cryptsetup and lvm2 hooks does not appear to work with my non-standard boot.

First couple reboots have been underwhelming. Cryptsetup still doesn't seem to be available in the initramdisk and I can't find any kind of a comprehensive doc on the hooks available by default.

Some "guides" written in various eras seem to think hooks in `/usr/share` won't get picked up and need to be copied into `/etc/initramfs-tools/hooks`. That seems a strange way to run a railroad but I guess I'll give it a shot. Man page lists

              cd tmp/initramfs
              gunzip -c /boot/initrd.img-2.6.18-1-686 | \
              cpio -i -d -H newc --no-absolute-filenames

as the way to inspect the initrd.

So some multiple problems remain.

I dumped the initrd to see what was what and no cryptsetup utility was included.

Copying the cryptsetup script to /etc/initramfs-tools didn't change that so I added some echoes to the hook and determined that despite the inclusion of the config flag, the "setup" variable was still no. hard coding that to _yes_ got me cryptsetup in the initrd, but on reboot it still didn't seem to try and take the cryptdevice from my kernel line.

after opening the device manually, I found that crypt isn't my only issue, lvm doesn't appear to have everything it needs to rescan after the crypted drive is opened and find the two volumes I need to boot.

But it's nearly two and probably time to stop for the night.

### Some kind of success

I still can't get initramfs tools to automatically run cryptsetup _or_ lvm. But when it finally drops to the fallback terminal I can run

```
(initramfs) cryptsetup open /dev/disk/by-partlabel/navi navi
(initramfs) lvm
> lvchange -a y navi
^D^D
```

and get to booting.

## Networking utilities

This dingus forgot to install `wireless-tools` and friends.

Back into chroot

```
# apt install wireless-tools wpasupplicant connman
```

I'm not married to connman, it works well enough in Arch but has its quirks. netctl was arch's default and I disliked its systemd integration (and flakiness) I do roam enough that I want some sort of help setting up wifi. I might see if netctl is available on debian or connman might work fine.

