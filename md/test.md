# ZFS replace OS Drive

## Assumptions
- Running Arch with ZFS on root
- One system disk
- Old disk connected to system
- New disk connected by USB
- New disk has not been used before, should otherwise be properly wiped

Example disk layout:
```bash
Device         Start        End    Sectors    Size     Type
disk-part2      2048    1050623    1048576    512M     EFI Syst
disk-part3   1050624    3147775    2097152      1G     Solaris
disk-part4   3147776 3900000000 3896852225    1.8T     Solaris
```

- disk-part3 is used as the boot pool (bpool)
- disk-part4 is used as the root pool (rpool) 
## Some heading

1. Boot into a arch live usb with zfs enabled

```bash
NEW_DISK=/dev/disk/by-id/usb-
```

```bash
OLD_DISK=/dev/disk/by-id/nvme-
```

```bash
# create partions on the new disk
fdsik ${NEW_DISK}

# sort of button-pushes
n -> 2 -> [ENTER] -> +512M
n -> 3 -> [ENTER] -> +1G
n -> 4 -> [ENTER] -> [A bit less than the full disk]
t -> 2 -> EFI
t -> 3 -> 161
t -> 4 -> 162
```

```bash
# import pools on old disk and make snapshots
mkdir -p /mnt/old
zpool import -R /mnt/old -d ${OLD_DISK}-part3 bpool oldbpool -N
zpool import -R /mnt/old -d ${OLD_DISK}-part4 rpool oldrpool -N

zfs snapshot -r oldbpool@newdisk
zfs snapshot -r oldrpool@newdisk
```

```bash
# create poools and copy data
mkdir -p /mnt/new

zpool create -f \
    -o ashift=12 \
    -o autotrim=on -d \
    -o cachefile=/etc/zfs/zpool.cache \
    -o feature@async_destroy=enabled \
    -o feature@bookmarks=enabled \
    -o feature@embedded_data=enabled \
    -o feature@empty_bpobj=enabled \
    -o feature@enabled_txg=enabled \
    -o feature@extensible_dataset=enabled \
    -o feature@filesystem_limits=enabled \
    -o feature@hole_birth=enabled \
    -o feature@large_blocks=enabled \
    -o feature@livelist=enabled \
    -o feature@lz4_compress=enabled \
    -o feature@spacemap_histogram=enabled \
    -o feature@zpool_checkpoint=enabled \
    -O devices=off \
    -O acltype=posixacl -O xattr=sa \
    -O compression=lz4 \
    -O normalization=formD \
    -O relatime=on \
    -O canmount=off -O mountpoint=/boot -R /mnt/new \
    bpool ${OLD_DISK}-part3
    
zpool create -f \
    -o ashift=12 \
    -o autotrim=on \
    -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
    -O compression=lz4 \
    -O normalization=formD \
    -O relatime=on \
    -O canmount=off -O mountpoint=/ -R /mnt/new \
    rpool ${DISK}-part4


umount -R /mnt/new
zfs umount bpool
zfs umount rpool

zpool import -R /mnt/new -d ${NEW_DISK}-part3 bpool -N
zpool import -R /mnt/new -d ${NEW_DISK}-part4 rpool -N

zfs send -vP oldbpool@newdisk | zfs receive bpool
zfs send -vP oldrpool@newdisk | zfs receive rpool

#efter det här stängde jag ned och växlade disk
```

```bash
# prepare the new disk for boot
zpool export oldbpool
zpool export oldrpool

zfs mount rpool/ROOT/arch
zfs mount bpool/BOOT/arch
zfs mount -a

zpool set cachefile=/etc/zfs/zpool.cache rpool
zpool set cachefile=/etc/zfs/zpool.cache bpool

rm /mnt/etc/zfs/zpool.cache
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache

arch-chroot /mnt/new

# assumes we have already added zfs stuff to /etc/mkinitcpio.conf
mkinitcpio -p linux

# chechk that boot is recognised as zfs
grub-probe /boot

DISK=/dev/disk/nvme-new_disk

mkdosfs -F 32 -s 1 -n EFI ${DISK}-part2

mkdir /boot/efi
echo /dev/disk/by-uuid/$(blkid -s UUID -o value ${DISK}-part2) \
     /boot/efi vfat defaults 0 0 >> /etc/fstab
vim /etc/fstab
# remove the entry for /boot/efi on the old disk

mount /boot/efi

# assumes that grub is already configed for zfs on root with no changes in layout
grub-mkconfig -o /boot/grub/grub.cfg

grub-install --target=x86_64-efi --efi-directory=/boot/efi \
             --bootloader-id=arch --recheck --no-floppy

# is this needed         
rm /etc/hostid
zgenhostid $(hostid)

# leave the chroot environment
exit
```

```bash
# unmount everything and shutdown
umount -R /mnt
zfs umount -a
zpool export -a

shutdown now
```

