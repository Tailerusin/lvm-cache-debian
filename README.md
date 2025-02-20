# lvm-cache-debian

LVM cache in Debian
From the man-pages: "The cache logical volume type uses a small and fast LV to improve the performance of a large and slow LV. It does this by storing the frequently used blocks on the faster LV. LVM refers to the small fast LV as a cache pool LV. The large slow LV is called the origin LV. Due to requirements from dm-cache (the kernel driver), LVM further splits the cache pool LV into two devices - the cache data LV and cache metadata LV. The cache data LV is where copies of data blocks are kept from the origin LV to increase speed. The cache metadata LV holds the accounting information that specifies where data blocks are stored (e.g. on the origin LV or on the cache data LV). Users should be familiar with these LVs if they wish to create the best and most robust cached logical volumes. All of these associated LVs must be in the same VG."

Assuming LVM is already setup in HDD (e.g. from anaconda) and SSD is untouched.

Create a physical volume for the SSD
# pvcreate /dev/sdX
e.g. pvcreate /dev/sdb, sdb being the SSD.

Extend your existing volume group to include the SSD
# vgextend VOLUME_GROUP /dev/sdX
anaconda names the volume group as fedora_HOSTNAME.

Create a cache metadata LV
# lvcreate -n meta -L YMB VOLUME_GROUP /dev/sdX
Where Y is 1000 times smaller than the cache data LV, with a minimum of 8MB. e.g. if you have a 32 GB SSD, Y = 32MB.

Create a cache data LV
# lvcreate -n cache -l Z VOLUME_GROUP /dev/sdX
Where Z is the size of your cache. Use 99%PVS to use 99% of the available physical volume size, i.e. mostly all of the SSD.

Create the cache pool LV
# lvconvert --type cache-pool --cachemode writeback --poolmetadata VOLUME_GROUP/meta VOLUME_GROUP/cache
This combines the cache data and metadata into a cache pool that uses writeback mode. Ommitting defaults to writethrough, which stores data in the cache and on the origin LV.

Create the cache
# lvconvert --type cache --cachepool VOLUME_GROUP/cache VOLUME_GROUP/root
The pool is then converted into a cache for the root partition.

Rebuild initramfs
First, install the "thin-provisioning-tools" package

Then, copy the following simple script into "/etc/initramfs-tools/hooks":

#!/bin/sh

PREREQ="lvm2"

prereqs()
{
    echo "$PREREQ"
}

case $1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

if [ ! -x /usr/sbin/cache_check ]; then
    exit 0
fi

. /usr/share/initramfs-tools/hook-functions

copy_exec /usr/sbin/cache_check

manual_add_modules dm_cache dm_cache_mq
update-initramfs -u
This will rebuild the initramfs to include the cache LV.
