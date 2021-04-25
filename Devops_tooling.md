# DevOps Tooling Website Solution

In this follow-up tutorial,  we'd introducing a single DevOps Tooling Solution that will consist of the following:

- Jenkins - free and open source automation server used to build CI/CD pipelines.
- Kubernetes - an open-source container-orchestration system for automating computer application deployment, scaling, and management.
- Jfrog Artifactory - Universal Repository Manager supporting all major packaging formats, build tools and CI servers. Artifactory.
- Rancher - an open source software platform that enables organizations to run and manage Docker and Kubernetes in production.
- Grafana - a multi-platform open source analytics and interactive visualization web application.
- Prometheus - An open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
- Kibana - Kibana is a free and open user interface that lets you visualize your Elasticsearch data and navigate the Elastic Stack.

In this project we'll implement a solution that consists of following components:

1. Infrastructure: GCP
2. Webserver Linux: Red Hat Enterprise Linux 8
3. Database Server: Ubuntu 20.04 + MySQL
4. Storage Server: Red Hat Enterprise Linux 8 + NFS Server
5. Programming Language: PHP

*Architectural image
## Step 1 - Prepare NFS Server
1. Spin up a new VM instance with RHEL Linux 8 Operating System.
2. Configure LVM on the Server (Follow the following steps to carry out the LVM config):
   
Run

`$ sudo lsblk`

```
Output:

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   10G  0 disk 
```

Then

`$ sudo gdisk /dev/sdb`

Verify, by running `$ sudo lsblk`

```
Output:

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   10G  0 disk 
└─sdb1   8:17   0   10G  0 part 
```

3. Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs.
   
   First create a Physical Volume (pv), then a volume group (vg) and then finally the Logical Volume (lv). Follow these steps to achieve that

   `sudo pvcreate /dev/sdb1`

   `sudo vgcreate vg-filestorage /dev/sdb1`

   then

   `sudo lvcreate -n lv-apps -L 3.5G vg-filestorage`

    `sudo lvcreate -n lv-logs -L 3.5G vg-filestorage`

     `sudo lvcreate -n lv-opt -L 2.5G vg-filestorage`

  verify by running `sudo lvs`

```
  lv-apps vg-filestorage -wi-a----- 3.50g                                           
         
  lv-logs vg-filestorage -wi-a----- 3.50g                                           
         
  lv-opt  vg-filestorage -wi-a----- 2.50g  
```

4. Formating the disks as  xfs

`$ sudo mkfs.xfs /dev/vg-filestorage/lv-apps
meta-data=/dev/vg-filestorage/lv-apps`

`$ sudo mkfs.xfs /dev/vg-filestorage/lv-apps
meta-data=/dev/vg-filestorage/lv-logs`

`$ sudo mkfs.xfs /dev/vg-filestorage/lv-apps
meta-data=/dev/vg-filestorage/lv-opt`
                       
5.  Create mount points on /mnt directory for the logical volumes as follow: Mount lv-apps on /mnt/apps - To be used by webservers Mount lv-logs on /mnt/logs - To be used by webserver logs Mount lv-opt on /mnt/opt              

```
$ sudo mkdir /mnt/apps
$ sudo mkdir /mnt/logs
$ sudo mkdir /mnt/opt

$ ls -l /mnt/
total 0
drwxr-xr-x. 2 root root 6 Apr 17 17:19 apps
drwxr-xr-x. 2 root root 6 Apr 17 17:19 logs
drwxr-xr-x. 2 root root 6 Apr 17 17:19 opt

$ sudo mount /dev/vg-filestorage/lv-apps /mnt/apps
$ sudo mount /dev/vg-filestorage/lv-logs /mnt/logs
$ sudo mount /dev/vg-filestorage/lv-opt /mnt/opt
```

verify by running `sudo lsblk` and `df -h`

`$ sudo lsblk`

```
Output:

NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0   20G  0 disk 
├─sda1                         8:1    0  200M  0 part /boot/efi
└─sda2                         8:2    0 19.8G  0 part /
sdb                            8:16   0   10G  0 disk 
└─sdb1                         8:17   0   10G  0 part 
  ├─vg--filestorage-lv--apps 253:0    0  3.5G  0 lvm  /mnt/apps
  ├─vg--filestorage-lv--logs 253:1    0  3.5G  0 lvm  /mnt/logs
  └─vg--filestorage-lv--opt  253:2    0  2.5G  0 lvm  /mnt/opt
```

`$ df -h`

```
Output:

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
```

6. Append these into the /etc/fstab

`$ sudo blkid`

```
Output:

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
```

```
$ sudo mount -a

$ systemctl daemon-reload
```

7. Install NFS server, configure it to start on reboot and make sure it is running.

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
```

`$ sudo systemctl enable nfs-server.service`

```
Output:

Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /us
r/lib/systemd/system/nfs-server.service.
```

`$ sudo systemctl status nfs-server.service`

```
Output:

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

8. Ensure you set up permission that will allow our Web servers to read, write and execute files on NFS:

```
$ sudo chown -R nobody: /mnt/apps
$ sudo chown -R nobody: /mnt/logs
$ sudo chown -R nobody: /mnt/opt
$ sudo chmod -R 777 /mnt/apps
$ sudo chmod -R 777 /mnt/logs
$ sudo chmod -R 777 /mnt/opt

$ ls -l /mnt
total 0
drwxrwxrwx. 2 nobody nobody 6 Apr 17 17:17 apps
drwxrwxrwx. 2 nobody nobody 6 Apr 17 17:18 logs
drwxrwxrwx. 2 nobody nobody 6 Apr 17 17:18 opt

$ sudo systemctl restart nfs-server.service
```

9. Configure access to NFS for clients within the same subnet (example of Subnet CIDR 

```
$ sudo vi /etc/exports
$ cat /etc/exports
/mnt/apps 10.154.0.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 10.154.0.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 10.154.0.0/20(rw,sync,no_all_squash,no_root_squash)

$ sudo exportfs -arv
exporting 10.154.0.0/20:/mnt/opt
exporting 10.154.0.0/20:/mnt/logs
exporting 10.154.0.0/20:/mnt/apps

$ sudo systemctl restart nfs-server.service
```

Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)

`$ rpcinfo -p | grep nfs`

```
Output:

    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
```



## Step 2 — Configure the database server

```
araflyayinde@db-mysql:~$ sudo mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 8.0.23-0ubuntu0.20.04.1 (Ubuntu)
Copyright (c) 2000, 2021, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement
.
mysql> create database tooling;
Query OK, 1 row affected (0.01 sec)
mysql> show datasbases;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual th
at corresponds to your MySQL server version for the right syntax to use near 
'datasbases' at line 1
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| tooling            |
+--------------------+
5 rows in set (0.00 sec)
mysql> CREATE USER 'webaccess'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.01 sec)
mysql> GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'%';
Query OK, 0 rows affected (0.01 sec)
mysql> exit
Bye
araflyayinde@db-mysql:~$ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
araflyayinde@db-mysql:~$ sudo systemctl restart mysql
araflyayinde@db-mysql:~$ 
```

## Step 3 — Prepare the Web Servers

```
Installed:
  apr-1.6.3-11.el8.x86_64                                                    
  apr-util-1.6.1-6.el8.x86_64                                                
  apr-util-bdb-1.6.1-6.el8.x86_64                                            
  apr-util-openssl-1.6.1-6.el8.x86_64                                        
  centos-logos-httpd-80.5-2.el8.noarch                                       
  httpd-2.4.37-30.module_el8.3.0+561+97fdbbcc.x86_64                         
  httpd-filesystem-2.4.37-30.module_el8.3.0+561+97fdbbcc.noarch              
  httpd-tools-2.4.37-30.module_el8.3.0+561+97fdbbcc.x86_64                   
  mailcap-2.1.48-3.el8.noarch                                                
  mod_http2-1.15.7-2.module_el8.3.0+477+498bb568.x86_64                      
Complete!
[araflyayinde@webserv-1 ~]$ sudo systemctl start httpd
[araflyayinde@webserv-1 ~]$ sudo systemctl enable httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /
usr/lib/systemd/system/httpd.service.
[araflyayinde@webserv-1 ~]$ ls -l /var/www
total 0
drwxr-xr-x. 2 root root 6 Nov  4 03:23 cgi-bin
drwxr-xr-x. 2 root root 6 Nov  4 03:23 html



[araflyayinde@webserv-1 ~]$ sudo ls -l /var/log/httpd
total 4
-rw-r--r--. 1 root root   0 Apr 19 21:26 access_log
-rw-r--r--. 1 root root 855 Apr 19 21:26 error_log
[araflyayinde@webserv-1 ~]$ sudo mv /var/log/httpd /var/log/httpdyke
[araflyayinde@webserv-1 ~]$ sudo ls -l /var/log/
total 844
drwx------. 2 root   root       23 Apr 17 14:48 audit
-rw-------. 1 root   root    10078 Apr 19 21:10 boot.log
-rw-------. 1 root   root    10771 Apr 18 03:23 boot.log-20210418
-rw-rw----. 1 root   utmp   135936 Apr 18 20:14 btmp
drwxr-xr-x. 2 chrony chrony      6 Nov 19  2019 chrony
-rw-------. 1 root   root     4934 Apr 19 21:10 cron
-rw-------. 1 root   root     5274 Apr 18 03:01 cron-20210418
-rw-------. 1 root   root    18822 Apr 19 21:25 dnf.librepo.log
-rw-r--r--. 1 root   root    22539 Apr 18 03:08 dnf.librepo.log-20210418
-rw-r--r--. 1 root   root    87753 Apr 19 21:25 dnf.log
-rw-r--r--. 1 root   root     6130 Apr 19 21:25 dnf.rpm.log
-rw-r-----. 1 root   root      372 Apr 19 21:10 firewalld
-rw-------. 1 root   root      714 Apr 19 21:25 hawkey.log
-rw-r--r--. 1 root   root      612 Apr 18 03:08 hawkey.log-20210418
drwx------. 2 root   root       41 Apr 19 21:26 httpdyke
-rw-rw-r--. 1 root   utmp   292292 Apr 19 21:25 lastlog
-rw-------. 1 root   root        0 Apr 18 03:23 maillog
-rw-------. 1 root   root        0 Mar 15 19:19 maillog-20210418
-rw-------. 1 root   root   209877 Apr 19 21:32 messages
-rw-------. 1 root   root   185098 Apr 18 03:18 messages-20210418
drwx------. 2 root   root        6 Mar 15 19:18 private
drwxr-xr-x. 2 root   root        6 Mar  3 17:12 qemu-ga
-rw-------. 1 root   root    25370 Apr 19 21:37 secure
-rw-------. 1 root   root    69542 Apr 18 03:03 secure-20210418
-rw-------. 1 root   root        0 Apr 18 03:23 spooler

Remember to change the user and group of /var/log/httpd from nobody to root chmod 700
[araflyayinde@webserv-1 ~]$ sudo ls -l /var/log/httpd
total 20
-rw-r--r--. 1 root root 12075 Apr 24 16:30 access_log
-rw-r--r--. 1 root root  4321 Apr 24 13:56 error_log
[araflyayinde@webserv-1 ~]$ systemctlrestart httpd
-bash: systemctlrestart: command not found
[araflyayinde@webserv-1 ~]$ sudo systemctl restart httpd

[araflyayinde@webserv-1 ~]$ git clone https://github.com/darey-io/tooling.git
Cloning into 'tooling'...
remote: Enumerating objects: 239, done.
remote: Total 239 (delta 0), reused 0 (delta 0), pack-reused 239
Receiving objects: 100% (239/239), 282.38 KiB | 5.13 MiB/s, done.
Resolving deltas: 100% (135/135), done.
[araflyayinde@webserv-1 ~]$ ls -l
total 0
drwxrwxr-x. 4 araflyayinde araflyayinde 191 Apr 19 21:49 tooling
```

## snag
```
[araflyayinde@webserv-1 ~]$ unmount -t nfs rw,nosuid 10.154.0.6:/mnt/logs
 /var/log/httpd
-bash: unmount: command not found
[araflyayinde@webserv-1 ~]$ df -h
Filesystem            Size  Used Avail Use% Mounted on
devtmpfs              1.9G     0  1.9G   0% /dev
tmpfs                 1.9G     0  1.9G   0% /dev/shm
tmpfs                 1.9G   17M  1.9G   1% /run
tmpfs                 1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2              20G  2.6G   18G  14% /
/dev/sda1             200M  6.9M  193M   4% /boot/efi
10.154.0.6:/mnt/apps  3.5G   58M  3.5G   2% /var/www
10.154.0.6:/mnt/logs  3.5G   57M  3.5G   2% /var/log/httpd
tmpfs                 374M     0  374M   0% /run/user/1000

[araflyayinde@webserv-1 ~]$ sudo umount -t nfs rw,nosuid 10.154.0.6:/mnt/
logs /var/log/httpd
umount: rw,nosuid: no mount point specified.
umount: /var/log/httpd: not mounted.
[araflyayinde@webserv-1 ~]$ df -h
Filesystem            Size  Used Avail Use% Mounted on
devtmpfs              1.9G     0  1.9G   0% /dev
tmpfs                 1.9G     0  1.9G   0% /dev/shm
tmpfs                 1.9G   17M  1.9G   1% /run
tmpfs                 1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2              20G  2.6G   18G  14% /
/dev/sda1             200M  6.9M  193M   4% /boot/efi
10.154.0.6:/mnt/apps  3.5G   58M  3.5G   2% /var/www
tmpfs                 374M     0  374M   0% /run/user/1000
[araflyayinde@webserv-1 ~]$ ls -l /var/www/html
total 40
-rw-r--r--. 1 root root 2909 Apr 19 21:57 admin_tooling.php
-rw-r--r--. 1 root root 1531 Apr 19 21:57 create_user.php
-rw-r--r--. 1 root root 4385 Apr 19 21:57 functions.php
drwxr-xr-x. 2 root root  183 Apr 19 21:57 img
-rw-r--r--. 1 root root 3162 Apr 19 21:57 index.php
-rw-r--r--. 1 root root  780 Apr 19 21:57 login.php
-rw-r--r--. 1 root root   19 Apr 19 21:57 README.md
-rw-r--r--. 1 root root 1097 Apr 19 21:57 register.php

```

Back in the nfs
```
[araflyayinde@nfs ~]$ ls -l /mnt/logs
total 8
-rw-r--r--. 1 root root 1485 Apr 19 21:55 access_log
-rw-r--r--. 1 root root 1133 Apr 19 21:55 error_log
[araflyayinde@nfs ~]$ sudo rm -rf /mnt/logs/access_log && sudo rm -rf /mnt/
logs/error_log 
[araflyayinde@nfs ~]$ ls -l /mnt/logs
total 0

```

## Webserver

```
[araflyayinde@webserv-1 ~]$ cd /var/www/html
[araflyayinde@webserv-1 html]$ ls
admin_tooling.php  img        README.md     tooling_stylesheets.css
create_user.php    index.php  register.php
functions.php      login.php  style.css
[araflyayinde@webserv-1 html]$ sudo vi functions.php 
```

## Mistake in tooling-db.sql
```

--
-- Dumping data for table `users`
--
INSERT INTO `users` (`id`, `username`, `password`, `email`, `user_type`, `s
tatus`) VALUES
(1, 'admin', '21232f297a57a5a743894a0e4a801fc3', 'dare@dare.com', 'admin', 
'1')


[araflyayinde@webserv-1 tooling]$ mysql -h 34.82.54.113 -u webaccess -p to
oling < tooling-db.sql
Enter password: 
ERROR 1064 (42000) at line 43: You have an error in your SQL syntax; check
 the manual that corresponds to your MySQL server version for the right sy
ntax to use near 'ALTER TABLE `users`
  ADD PRIMARY KEY (`id`)' at line 11



[araflyayinde@webserv-1 tooling]$ sudo vi tooling-db.sql 
[araflyayinde@webserv-1 tooling]$ mysql -h 34.82.54.113 -u webaccess -p too
ling < tooling-db.sql
Enter password: 
ERROR 1050 (42S01) at line 30: Table 'users' already exists
[araflyayinde@webserv-1 tooling]$ mysql -h 34.82.54.113 -u webaccess -p too
ling < tooling-db.sql
Enter password: 
[araflyayinde@webserv-1 tooling]$ ls
apache-config.conf  html         README.md     tooling-db.sql
Dockerfile          Jenkinsfile  start-apache
[araflyayinde@webserv-1 tooling]$ sudo setsebool -P httpd_can_network_conne
ct 1
[araflyayinde@webserv-1 tooling]$ sudo setsebool -P http_use_nfs 1
Boolean http_use_nfs is not defined
[araflyayinde@webserv-1 tooling]$ sudo setsebool -P httpd_use_nfs 1
[araflyayinde@webserv-1 tooling]$ sudo setsebool -P httpd_can_network_conne
ct_db 1
[araflyayinde@webserv-1 tooling]$ cd ..
[araflyayinde@webserv-1 ~]$ ls -l /var/www/html/
total 40
-rw-r--r--. 1 root root 2909 Apr 19 21:57 admin_tooling.php
-rw-r--r--. 1 root root 1531 Apr 19 21:57 create_user.php
-rw-r--r--. 1 root root 4373 Apr 23 04:38 functions.php
drwxr-xr-x. 2 root root  183 Apr 19 21:57 img
-rw-r--r--. 1 root root 3162 Apr 19 21:57 index.php
-rw-r--r--. 1 root root  780 Apr 19 21:57 login.php
-rw-r--r--. 1 root root   19 Apr 19 21:57 README.md
-rw-r--r--. 1 root root 1097 Apr 19 21:57 register.php
-rw-r--r--. 1 root root 1704 Apr 19 21:57 style.css
-rw-r--r--. 1 root root 1027 Apr 19 21:57 tooling_stylesheets.css
[araflyayinde@webserv-1 ~]$ cat /var/www/html/index.php

```