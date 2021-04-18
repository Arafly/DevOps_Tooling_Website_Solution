## Step 1 - Prepare NFS Server
Spin up a new EC2 instance with RHEL Linux 8 Operating System.
Configure LVM on the Server.

```
[araflyayinde@nfs ~]$ sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   10G  0 disk 
[araflyayinde@nfs ~]$ sudo gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.3
Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present
Creating new GPT entries.

Changed type of partition to 'Linux LVM'
Command (? for help): p
Disk /dev/sdb: 20971520 sectors, 10.0 GiB
Model: PersistentDisk  
Sector size (logical/physical): 512/4096 bytes
Disk identifier (GUID): 381F6626-401E-4BA4-8D38-341EFAA9A7E4
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 20971486
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20971486   10.0 GiB    8E00  Linux LVM
Command (? for help): w
Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!
Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sdb.
The operation has completed successfully.
[araflyayinde@nfs ~]$ sudo gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.3
Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present
Found valid GPT with protective MBR; using GPT.
Command (? for help): ^C
[araflyayinde@nfs ~]$ sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   10G  0 disk 
└─sdb1   8:17   0   10G  0 part 


c Convert
  lv-apps vg-filestorage -wi-a----- 3.50g                                           
         
  lv-logs vg-filestorage -wi-a----- 3.50g                                           
         
  lv-opt  vg-filestorage -wi-a----- 2.50g                                           
         
[araflyayinde@nfs ~]$ sudo mkfs.xfs /dev/vg-filestorage/lv-apps
meta-data=/dev/vg-filestorage/lv-apps isize=512    agcount=4, agsize=229376 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=917504, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
[araflyayinde@nfs ~]$ sudo mkfs.xfs /dev/vg-filestorage/lv-logs
meta-data=/dev/vg-filestorage/lv-logs isize=512    agcount=4, agsize=229376 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=917504, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
[araflyayinde@nfs ~]$ sudo mkfs.xfs /dev/vg-filestorage/lv-opt
meta-data=/dev/vg-filestorage/lv-opt isize=512    agcount=4, agsize=163840 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=655360, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.


[araflyayinde@nfs ~]$ sudo mkdir /mnt/apps
[araflyayinde@nfs ~]$ sudo mkdir /mnt/logs
[araflyayinde@nfs ~]$ sudo mkdir /mnt/opt
[araflyayinde@nfs ~]$ ls -l
total 0
[araflyayinde@nfs ~]$ ls -l /mnt/
total 0
drwxr-xr-x. 2 root root 6 Apr 17 17:19 apps
drwxr-xr-x. 2 root root 6 Apr 17 17:19 logs
drwxr-xr-x. 2 root root 6 Apr 17 17:19 opt
[araflyayinde@nfs ~]$ sudo mount /dev/vg-filestorage/lv-apps /mnt/apps
[araflyayinde@nfs ~]$ sudo mount /dev/vg-filestorage/lv-logs /mnt/logs
[araflyayinde@nfs ~]$ sudo mount /dev/vg-filestorage/lv-opt /mnt/opt
[araflyayinde@nfs ~]$ sudo sblk
sudo: sblk: command not found
[araflyayinde@nfs ~]$ sudo lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0   20G  0 disk 
├─sda1                         8:1    0  200M  0 part /boot/efi
└─sda2                         8:2    0 19.8G  0 part /
sdb                            8:16   0   10G  0 disk 
└─sdb1                         8:17   0   10G  0 part 
  ├─vg--filestorage-lv--apps 253:0    0  3.5G  0 lvm  /mnt/apps
  ├─vg--filestorage-lv--logs 253:1    0  3.5G  0 lvm  /mnt/logs
  └─vg--filestorage-lv--opt  253:2    0  2.5G  0 lvm  /mnt/opt
[araflyayinde@nfs ~]$ df -h
Filesystem                            Size  Used Avail Use% Mounted on
devtmpfs                              1.9G     0  1.9G   0% /dev
tmpfs                                 1.9G     0  1.9G   0% /dev/shm
tmpfs                                 1.9G  8.4M  1.9G   1% /run
tmpfs                                 1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2                              20G  2.5G   18G  13% /
/dev/sda1                             200M  6.9M  193M   4% /boot/efi
tmpfs                                 374M     0  374M   0% /run/user/1000
/dev/mapper/vg--filestorage-lv--apps  3.5G   58M  3.5G   2% /mnt/apps
/dev/mapper/vg--filestorage-lv--logs  3.5G   58M  3.5G   2% /mnt/logs
/dev/mapper/vg--filestorage-lv--opt   2.5G   51M  2.5G   2% /mnt/opt

# After editing this file, run 'systemctl daemon-reload' to update systemd
[araflyayinde@nfs ~]$ sudo blkid
/dev/sda2: LABEL="root" UUID="df262b9a-97e1-403f-80e5-5f57eb344195" BLOCK_SIZE="4096
" TYPE="xfs" PARTUUID="36a2f04d-1e93-4b2a-90d0-ef3e99be94b7"
/dev/sda1: SEC_TYPE="msdos" UUID="2D71-C36C" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL=
"EFI System Partition" PARTUUID="e9aa0136-c3dc-4cef-9923-a2988ccf8dff"
/dev/sdb1: UUID="fvKKIi-TVNp-1dh1-jz5s-fqbZ-7k3S-0H660L" TYPE="LVM2_member" PARTLABE
L="Linux LVM" PARTUUID="79e0eead-b2df-48aa-bf36-d5710b643d8b"
/dev/mapper/vg--filestorage-lv--apps: UUID="bd4728e5-7d42-430e-836b-c934f4cfdc5c" BL
OCK_SIZE="4096" TYPE="xfs"
/dev/mapper/vg--filestorage-lv--logs: UUID="7aca6be8-8b31-4522-9ea7-35b002dcc9b5" BL
OCK_SIZE="4096" TYPE="xfs"
/dev/mapper/vg--filestorage-lv--opt: UUID="3090d76f-fc1d-4ad0-9ab1-3248da27e044" BLO
CK_SIZE="4096" TYPE="xfs"
[araflyayinde@nfs ~]$ sudo vi /etc/fstab
[araflyayinde@nfs ~]$ sudo vi /etc/fstab
[araflyayinde@nfs ~]$ sudo vi /etc/fstab
[araflyayinde@nfs ~]$ sudo mount -a
[araflyayinde@nfs ~]$ sudo vi /etc/fstab
[araflyayinde@nfs ~]$ systemctl daemon-reload

[araflyayinde@nfs ~]$ sudo systemctl enable nfs-server.service
Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /us
r/lib/systemd/system/nfs-server.service.
[araflyayinde@nfs ~]$ sudo systemctl status nfs-server.service
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor pres>
   Active: active (exited) since Sat 2021-04-17 17:54:00 UTC; 22s ago
 Main PID: 143134 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 23411)
   Memory: 0B
   CGroup: /system.slice/nfs-server.service
Apr 17 17:54:00 nfs systemd[1]: Starting NFS server and services...
Apr 17 17:54:00 nfs systemd[1]: Started NFS server and services.
```