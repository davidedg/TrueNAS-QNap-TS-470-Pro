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

    sudo zpool create ssd mirror /dev/disk/by-partuuid/5a76cf19-ba92-43ee-81ab-fcf287ea2e57 /dev/disk/by-partuuid/cf936eda-a2ba-4030-9294-db2f78c85feb
    sudo zpool export ssd

(please note the below error message is expected and ok):

    cannot mount '/ssd': failed to create mountpoint: Read-only file system


In the GUI, import the new created pool

\
NTFS-3G:
--------

[https://gist.github.com/davidedg/5c85dd138cfd6a870f5784e3e123decd#file-true-nas-scale-ntfs-install-md](https://gist.github.com/davidedg/5c85dd138cfd6a870f5784e3e123decd#file-true-nas-scale-ntfs-install-md)

