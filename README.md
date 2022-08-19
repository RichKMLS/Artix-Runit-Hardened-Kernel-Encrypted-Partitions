These are my (literal) hand-written steps for installing Artix linux. <br>
Some steps may be outdated or I may have mistakingly transcribed the step wrong.
<br>

A few things to keep in mind before copy pasting everything here
<br>

- This was written around the time artools-chroot became artix-chroot so it is a few years old.<br>(logical volume parts are a slightly newer addition)
- These steps assume you have already installed 'artix-base-runit-xxxxxx.iso' from https://artixlinux.org/download.php and have flashed the image to a usb and you are now at the process of installing artix onto a hard drive.
- These steps also make some assumptions about your linux knowledge and doesn't hold your hand. If you don't know what a command does please consider looking it up :)
- These steps were not made with the intention of sharing them online so it may seem like I skipped some things.
- Next time I install a Artix on a new machine I will go over these steps to see if they are still valid (whenever that could be) and will document the steps fully at that time (and hopefully improve them).
- This is for education purposes only :) -> use the offical guide https://wiki.artixlinux.org/Main/Installation


---

<br>
<br>

# Artix Runit Hardened-Kernel <br>Encrypted Partitions and Logical Volume

<br>

fdisk /dev/sdx #create partitions

#--------------------if creating logical volume--------------------<br>
vgcreate [groupName] /dev/sdx# /dev/sdy#<br>
lvcreate -l 100%FREE -n [volName] [groupName]<br>
#-------------------------------------------------------------------------

cryptsetup open --type plain -d /dev/urandom /dev/sdx# wipe#
dd bs=1M if=/dev/urandom of=/dev/mapper/wipe# status=progress #overkill?

cryptsetup -v --type luks2 -c aes-xts-plain64 -s 512 -h sha512 -i 5000 --use-random -y luksFormat /dev/sdx#

#--------------------if creating logical volume--------------------<br>
#replace '/dev/sdx#' with '/dev/[groupName]/[volName]' <br>
#in the command above and below if using logical volume<br>
#-------------------------------------------------------------------------

cryptsetup open /dev/sdx# [cryptvolume]

mkfs.ext4 /dev/mapper/[cryptvolume]
mkswap /dev/mapper/[cryptSwap]
mkfs.fat -F32 /dev/[boot partition (sdx1)]

swapon /dev/mapper/[cryptSwap]<br>
mount /dev/mapper/[cryptRoot] /mnt<br>
mkdir /mnt/home /mnt/boot<br>
mount /dev/mapper/[cryptHome] /mnt/home<br>
mount /dev/mapper[cryptOther] ...  #mount the rest<br>
mount /dev/sdx1 /mnt/boot

basestrap /mnt base base-devel runit linux-hardened linux-firmware elogind-runit networkmanager-runit cryptsetup-runit grub efibootmgr vim ranger 

#--------------------if creating logical volume--------------------<br>
#add lvm2 to the basestrap command<br>
#-------------------------------------------------------------------------

fstabgen -U /mnt >> /mnt/etc/fstab

sed -i "s%quiet%quiet cryptdevice=/dev/disk/by-uuid/$(lsblk -o +UUID | grep sdx2 | awk '{print $NF}'):[cryptRoot]%g" /mnt/etc/default/grub

echo -e "[cryptSwap]\tUUID=$=$(lsblk -o +UUID | grep [root partition (sdx2)] | awk '{print $NF}')\t/etc/KeyFile" >> /mnt/etc/crypttab

#repeat the previous command with [cryptHome], [cryptOther] and associated paritions

artix-chroot /mnt

ln -sf /usr/share/zoneinfo/[country]/[city] /etc/localtime

hwclock --systohc

echo -e 'export LANG="en_US.UTF-8"\nexport LC_COLLATE="c"' >> /etc/locale.conf

sed -i s/#en_US/en_US/g /etc/locale-gen

locale-gen

echo "[hostname]" >> /etc/hostname

vim /etc/hosts

    127.0.0.1        localhost
    ::1              localhost
    127.0.0.1        [hostname].localdomain        [hostname]
    
vim /etc/mkinitcpio.conf

    #add encrypt after udev
    #if logical volume: add lvm2 after block

mkinitcpio -p linux-hardened

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

grub-mkconfig -o /boot/grub/grub.cfg

dd bs=512 count=4 if=/dev/urandom of=/etc/KeyFile iflag=fullblock

chmod 600 /etc/KeyFile

cryptsetup luksAddKey /dev/[swap partition (sdx3)] /etc/KeyFile<br>
#(and [home partition (sdx4)] + cryptother)

ln -s /etc/runit/sv/NetworkManager/ /etc/runit/runsvdir/current

passwd

useradd -m -G wheel [username]

passwd [username]

exit<br>
exit<br>
poweroff now<br>
#remove usb and turn on pc<br>

---
  
    










