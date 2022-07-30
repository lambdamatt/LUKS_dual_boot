# Dual Boot 20.04 and 18.04 with LUKS
##1
![](1.png)
Format the SATA device and create 5 new partitions:
* Part1 will be the EFI boot partition
* Part2 will be the 20.04 /boot partition
* Part3 will be the 20.04 LUKS partition
* Part4 will be the 18.04 /boot partition
* Part5 will be the 18.04 LUKS partition

 Partition 1 should be 2GB
 The boot partitions should be big enough to hold all of the kernels that will ever be installed 10GB is very safe
 The LUKS partitions should half of the total drive capacity each, less the 2GB used for EFI and the amount used for the boot partitions.

**NOTE--** These will all be formatted later on. **DO NOT** select the encryption options. Use defaults for now

##2
![](2.png)
This guide assumes that the drive is labeled as SDA. This is what the output of `lsblk /dev/sda` should look like

##3
![](3.png)
First we will setup /dev/sda1 for use as the EFI partition

`mkfs.vfat -F 16 -n EFI-SP /dev/sda1`

##4
![](4.png)
Now we will build the LUKS contianer for the 20.04 root partition

`cryptsetup luksFormat /dev/sda3`

##5
![](5.png)
Now we will decrypt and map /dev/sda3 to sda3_crypt

`cryptsetup open /dev/sda3 sda3_crypt`

`ls /dev/mapper` shows that is is mapped

##6
![](6.png)
Format /dev/sda2 as ext4 

`mkfs.ext4 L boot_2004 /dev/sda2`

##7
![](7.png)
We will use LVM under the LUKS partition. This cleared up the major issues I was having in other testing and all of the documentation pointed to use LVM


```
pvcreate/dev/mapper/sda3_crypt
vgcreate 2004-vg /dev/mapper/sda3_crypt
lvcreate -L 4G -n swap_1 2004 vg
lvcreate -l 80%FREE -n root 2004-vg
```

```
Block diagram of LUKS and LVM
  ┌─────────────────────────────────────┐
  │     LUKS                            │
  │ ┌─────────────────────────────────┐ │
  │ │   PV                            │ │
  │ │ ┌─────────────────────────────┐ │ │
  │ │ │ GV                          │ │ │
  │ │ ├───────┬─────────────────────┤ │ │
  │ │ │ LV    │        LV           │ │ │
  │ │ │       │                     │ │ │
  │ │ │   SWAP│          ROOT       │ │ │
  │ │ │       │                     │ │ │
  │ │ │       │                     │ │ │
  │ │ │       │                     │ │ │
  │ │ │       │                     │ │ │
  │ ├─┼───────┼─────────────────────┼─┤ │
  └─┴─────────────────────────────────┴─┘
```

##8
![](8.png)

##9 
![](9.png)
Launch the Installer and choose 'Something else' as the Installation type

##10
![](10.png)
Select the child /dev/mapper/2004-vg-root and edit the partition as EXT4, format the partition, and set '/' as the mount point

##11
![](11.png)
Select the child /dev/mapper/2004-vg-swap_1 and 'Use as: swap area'

##12
![](12.png)
Select /dev/sda1 and 'Use as: EFI System Partition'

##13
![](13.png)
Select /dev/sda2 and Format as EXT4, format the partition and set mount point as '/boot'

##14
![](14.png)
Select the root device-- /dev/sda as the bootloader installation device

##15
![](15.png)

##16
![](16.png)

##17
![](17.png)
After Installtion is complete, choose Continue Testing

##18
![](18.png)
We need to chroot into the new install so we have to mount the proc sys dev and etc/resolv devices onto the new install before we reboot

`for n in proc sys dev etc/resolv.conf; do mount --rbind /$n /target/$n; done`
`chroot /target`
`mount -av`

you should see:

```
/
/boot
/boot/efi
```

##19
![](19.png)
Create a crypttab file
sda3_crypt is the mapper lable for sda3, the UUID should be for /dev/sda3 and 'none luks,discard' options are added
This one-liner will set it up correctly, assuming again that the device is /dev/sda

`echo "sda3_crypt UUID=$(blkid -s UUID -o value /dev/sda3) none luks,discard" > /etc/crypttab`

##20 
![](20.png)
edit /etc/default/grub to show the grub menu and rebuild grub
I neglected to update-initramfs before I rebooted
**DO NOT REBOOT until you have run `update-initramfs -u -k all`** [shown on slide 24](#24)

##21
![](21.png)
I inclued these and the next several slides to show you how to back out of the mistake I made, I thought it might be helpful
You can see here that /dev/mapper doesn't have my LVM volume group because GRUB did not prompt to decrypt the partition

##22
![](22.png)
From the initramfs busybox environment
`cryptsetup open /dev/sda3 sda3_crypt`
this will map the the expected mapper location and boot can resume

##23
![](23.png)
`exit` to continue boot

##24
![](24.png)
This is what I should have done on slide 20

##25 
![](25.png)
Reboot and you should be prompted to unlock sda3_crypt

##26
![](26.png)
it works

##27 reboot into the UEFI/BIOS and boot from the 18.04 Rescue ISO
![](27.png)

##28
![](28.png)
similar to what we did for 20.04, but this time we will be using /dev/sda4 for /boot and /dev/sda5 for the LUKS partition

`cryptsetup luksFormat /dev/sda5`

##29
![](29.png)
open and setup the device mapper to sda5_crypt

`cryptsetup open /dev/sda5 sda5_crypt`

##30
![](30.png)
create an ext4 filesystem on /dev/sda4

`mkfs.ext4 -L boot_1804 /dev/sda4`

##31
![](31.png)
NOTE-- I messed up the LVM naming convention here. What I did works, but 100% of the documentation uses <<name>>--vg like i did for 20.04. You may want to sub 1804-vg for luks-1804 in the following commands.
```
pvcreate/dev/mapper/sda5_crypt
vgcreate luks-1804 /dev/mapper/sda5_crypt
lvcreate -L 4G -n swap_1 luks-1804
lvcreate -l 80%FREE -n root luks-1804
```

**CORRECTED**

```
vgcreate 1804-vg /dev/mapper/sda5_crypt
lvcreate -L 4G -n swap_1 1804-vg
lvcreate -l 80%FREE -n root 1804-vg
```

##32
![](32.png)
launch unquity without the installing a bootloader 

`uquiquity -b`

##33
![](33.png)
Installation type: Something else

##34
![](34.png)
/dev/mapper/luks--1804-root as EXT4, format, Mount Point "/"

##35
![](35.png)
/dev/mapper/luks--1804-swap_1 as 'swap area'

##36
![](36.png)
Check that /dev/sda is used as EFI System Partion. No change should be needed here

##37
![](37.png)
Edit /dev/sda for EXT4, format and mount to '/boot'

##38
![](38.png)
Install and continue testing upon completion

##39
![](39.png)
18.04 installer does not handle the LVM to /target mount as well as 20.04 so I ran

`mount /dev/mapper/luks--1804-root /target`

to mount it /target
then:

```
for n in proc sys dev etc/resolv.conf; do mount --rbind /$n /target/$n; done
chroot /target
mount -av
```

Then, very similar to for 20.04, just shifting from sda3 to sda5, setup /etc/crypttab
`echo "sda5_crypt UUID=$(blkid -s UUID -o value /dev/sda5) none luks,discard" > /etc/crypttab`

##40
![](40.png)
`update-initramfs -u -k all`
so that the initramfs will know about the crypttab
** we are relying on 20.04 GRUB so you do not need to update GRUB **

##41
![](41.png)
This is the GRUB menu for 20.04-- It doesn't know about 18.04 yet so we have one more cleanup task to perform. Boot into 20.04

##42
![](42.png)

##43
![](43.png)

update-grub is not finding the initrd images for the 5.4 kernel. Rather than forcing it to, we are going to expose the 18.04 install and run again so it sees it as a seperate OS

##44
![](44.png)

```
** NOT SHOWN ON SLIDE ** 
decrypt /dev/sda5 and map to expected label of sda5_crypt
cryptsetup open /dev/sda5 sda5_crypt 
```
activate root lv

```
lvchange -ay luks-1804
```

mount the lv to /mnt

```
mount /dev/mapper/luks--1804-root /mnt
```
##45 
![](45.png)
Run `update_grub` and it will find 18.04.6 and add a boot menu entry. 

##46
![](46.png)
18.04 shows up now

##47 
![](47.png)
Prompt to unlock sda5_crypt

##48
![](48.png)
booted into 18.04
![](49![]([](![](![](![]())))).png)

























