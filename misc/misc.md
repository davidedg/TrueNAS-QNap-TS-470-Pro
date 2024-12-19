INSTALL SSD SPLIT:
------------------

From this [Reddit post](https://www.reddit.com/r/truenas/comments/lgf75w/scalehowto_split_ssd_during_installation/)
\
\
Up to 24.04:

    sed -i 's/sgdisk -n3:0:0/sgdisk -n3:0:+16384M/g' /usr/sbin/truenas-install

24.10:

    sed -i 's/-n3:0:0/-n3:0:+16384M/g' /usr/lib/python3/dist-packages/truenas_installer/install.py

\
After the installation, inspect the partitions (here we used /dev/sda and /dev/sdb):

    truenas_admin@truenas[~]$ sudo partx -v -s  /dev/sda
    partition: none, disk: /dev/sda, lower: 0, upper: 0
    /dev/sda: partition table type 'gpt' detected
    range recount: max partno=3, lower=0, upper=0
    NR   START      END  SECTORS SIZE NAME UUID
     1    4096     6143     2048   1M      34cabe56-0b29-4467-994a-ae9ed5349f05
     2    6144  1054719  1048576 512M      8ae0e864-067a-4008-a799-4cc58892867d
     3 1054720 34609151 33554432  16G      7618a31e-686e-4354-b3ef-ed1c17e83339
    
    truenas_admin@truenas[~]$ sudo partx -v -s  /dev/sdb
    partition: none, disk: /dev/sdb, lower: 0, upper: 0
    /dev/sdb: partition table type 'gpt' detected
    range recount: max partno=3, lower=0, upper=0
    NR   START      END  SECTORS SIZE NAME UUID
     1    4096     6143     2048   1M      68ae43bd-bd1b-4dd8-948d-21977a1b7189
     2    6144  1054719  1048576 512M      87322d69-d433-4698-8e01-dd1025577a54
     3 1054720 34609151 33554432  16G      8e0535e5-c464-4ba9-b9f8-82d3cab1c33a



Create partitions (80GB - we're leaving ~32 GB for later use or for overprovisioning as these are brand new disks):

    sudo sgdisk -n 4:0:+80G -t4:BF01 /dev/sda
    sudo sgdisk -n 4:0:+80G -t4:BF01 /dev/sdb
    sudo partprobe /dev/sda
    sudo partprobe /dev/sdb

Inspect again, take note of the UUIDs of the new partitions (also visible with fdisk -lx /dev/sda):

    root@truenas[~]# sudo partx -v -s  /dev/sda
    partition: none, disk: /dev/sda, lower: 0, upper: 0
    /dev/sda: partition table type 'gpt' detected
    range recount: max partno=4, lower=0, upper=0
    NR    START       END   SECTORS SIZE NAME UUID
     1     4096      6143      2048   1M      34cabe56-0b29-4467-994a-ae9ed5349f05
     2     6144   1054719   1048576 512M      8ae0e864-067a-4008-a799-4cc58892867d
     3  1054720  34609151  33554432  16G      7618a31e-686e-4354-b3ef-ed1c17e83339
     4 34609152 202381311 167772160  80G      5a76cf19-ba92-43ee-81ab-fcf287ea2e57
    root@truenas[~]# sudo partx -v -s  /dev/sdb
    partition: none, disk: /dev/sdb, lower: 0, upper: 0
    /dev/sdb: partition table type 'gpt' detected
    range recount: max partno=4, lower=0, upper=0
    NR    START       END   SECTORS SIZE NAME UUID
     1     4096      6143      2048   1M      68ae43bd-bd1b-4dd8-948d-21977a1b7189
     2     6144   1054719   1048576 512M      87322d69-d433-4698-8e01-dd1025577a54
     3  1054720  34609151  33554432  16G      8e0535e5-c464-4ba9-b9f8-82d3cab1c33a
     4 34609152 202381311 167772160  80G      cf936eda-a2ba-4030-9294-db2f78c85feb

    
Then you can create and export the pool:

    sudo zpool create -o ashift=12 -o autoexpand=on ssd mirror /dev/disk/by-partuuid/5a76cf19-ba92-43ee-81ab-fcf287ea2e57 /dev/disk/by-partuuid/cf936eda-a2ba-4030-9294-db2f78c85feb
    sudo zpool export ssd

(please note the below error message is expected and ok):

    cannot mount '/ssd': failed to create mountpoint: Read-only file system


In the GUI, import the new created pool


REPLACE A FAILED DRIVE IN AN SSD SPLIT CFG:
-------------------------------------------

Let's assume sdb is the failed drive (but already replaced) and sda is the remaining disk.
\
Also, note that the uuids here will not match the uuids in the previous section, as I documented this for another system.

\
\
System -> Boot -> Boot Pool Status -> boot-pool -> mirror-0
\
Select the failed disk -> 3 dots -> Replace -> pick the new empty disk (sdb)

\
this procedure will ensure it is done the TrueNAS way, which will include cloning the EFI partition (/dev/sdX2) and re-installing the boot loader

\
However, the boot pool partition will be created using all the available space, i.e.:

    root@truenas[~]# sudo partx -v -s  /dev/sdb
    partition: none, disk: /dev/sdb, lower: 0, upper: 0
    /dev/sdb: partition table type 'gpt' detected
    range recount: max partno=3, lower=0, upper=0
    NR   START       END   SECTORS   SIZE NAME UUID
     1      40      2087      2048     1M      f1715c51-1a12-4f36-8695-bd8d6163a3a2
     2    2088   1050663   1048576   512M      f3823c0e-0451-468e-9ebe-c1e31bd7d05b
     3 1050664 268435422 267384759 127.5G      de3d5435-58d5-4eca-95c0-4c6a73ad37c5

\
To restore the ssd-split configuration, we'll need to adjust the partitions manually:

\
First, detach the new disk from the boot-pool volume:

    zpool detach boot-pool sdb3

Then delete the 3rd partition:

    sgdisk -d 3 /dev/sdb

Recreate 3rd and 4th partitions as per this initial guide:

    sgdisk -n 3:0:+16G -t3:BF01 /dev/sdb
    sgdisk -n 4:0:+80G -t4:BF01 /dev/sdb
    partprobe /dev/sdb

Check the sizes and record the UUID for the 4th partition:

    root@truenas[~]# sudo partx -v -s  /dev/sdb
    partition: none, disk: /dev/sdb, lower: 0, upper: 0
    /dev/sdb: partition table type 'gpt' detected
    range recount: max partno=4, lower=0, upper=0
    NR    START       END   SECTORS SIZE NAME UUID
     1       40      2087      2048   1M      f1715c51-1a12-4f36-8695-bd8d6163a3a2
     2     2088   1050663   1048576 512M      f3823c0e-0451-468e-9ebe-c1e31bd7d05b
     3  1050664  34605095  33554432  16G      09f91a0e-697b-4c30-9363-f3f6480e1cd8
     4 34605096 202377255 167772160  80G      c1e39c0c-022b-4a61-8f15-cdc092f1f699

(fun fact I noticed: the gui will re-create the partitions not aligned. I do not care too much as this is the boot volume).

\
Then re-attach the sdb3 to the boot-pool:

    zpool attach boot-pool /dev/sda3 /dev/sdb3

And wait for resilvering to finish

\
Check SSD pool and notice the id of the failed disk:

    root@truenas[~]# zpool status ssd
      pool: ssd
     state: DEGRADED
    status: One or more devices could not be used because the label is missing or
            invalid.  Sufficient replicas exist for the pool to continue
            functioning in a degraded state.
    action: Replace the device using 'zpool replace'.
       see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-4J
    config:
    
            NAME                                      STATE     READ WRITE CKSUM
            ssd                                       DEGRADED     0     0     0
              mirror-0                                DEGRADED     0     0     0
                13ed13d5-e4eb-429d-93f3-fc780011c06c  ONLINE       0     0     0
                10983865452315239826                  UNAVAIL      0     0     0  was /dev/disk/by-partuuid/b856820d-4f3b-459f-abee-af827f432b52



Finally, replace the failed disk with the new partition, using its uuid:

    zpool replace ssd 10983865452315239826 /dev/disk/by-partuuid/c1e39c0c-022b-4a61-8f15-cdc092f1f699


Check the status and wait until the resilver fully completes:

    root@truenas[~]# zpool status ssd
      pool: ssd
     state: ONLINE
      scan: resilvered 232M in 00:00:01 with 0 errors on Thu Dec 19 12:52:46 2024
    config:
    
            NAME                                      STATE     READ WRITE CKSUM
            ssd                                       ONLINE       0     0     0
              mirror-0                                ONLINE       0     0     0
                13ed13d5-e4eb-429d-93f3-fc780011c06c  ONLINE       0     0     0
                c1e39c0c-022b-4a61-8f15-cdc092f1f699  ONLINE       0     0     0
    
    errors: No known data errors
    

Immediately reboot after that so that the TrueNAS database can update its ids.


\
NTFS-3G:
--------

[https://gist.github.com/davidedg/5c85dd138cfd6a870f5784e3e123decd#file-true-nas-scale-ntfs-install-md](https://gist.github.com/davidedg/5c85dd138cfd6a870f5784e3e123decd#file-true-nas-scale-ntfs-install-md)

