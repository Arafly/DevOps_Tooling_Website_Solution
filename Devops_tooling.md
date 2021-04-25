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

![](https://github.com/Arafly/DevOps_Tooling_Website_Solution/blob/master/assets/Tooling-Website-Infrastructure.png)
## Step 1 - Prepare the NFS Server
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

6. Ensure that the changes can persist after reboot by appending these into the /etc/fstab

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

> In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049


## Step 2 — Configure the database server

The following steps are required to to fully set-up our DB server

- Install MySQL server
Create a database and name it *tooling*
- Create a database user with name *webaccess* and grant permission to the user on tooling db to be able to do anything only from the webservers subnet cidr

```
$ sudo mysql
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

~$ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
~$ sudo systemctl restart mysql
```
## Step 3 — Prepare the Web Servers

We need to make sure that our Web Servers can serve the same content from shared storage solutions, in our case - NFS Server and MySQL database.  For storing shared files that our Web Servers will use - we will utilize NFS and mount previously created Logical Volume lv-apps to the folder where Apache stores files to be served to the users (/var/www).

The following steps are required to make this possible.

1. Launch a new VMinstance with RHEL 8 Operating System
2. Install NFS client

`sudo yum install nfs-utils nfs4-acl-tools -y`

3. Mount /var/www/ and target the NFS server’s export for apps

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```

4. Verify that NFS was mounted successfully by running `df -h`
Also make sure that the changes will persist on Web Server after reboo, by adding following line:

`sudo vi /etc/fstab`

`<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0`

5. Install Apache

```
$ sudo yum install httpd -y

$ sudo systemctl start httpd
$ sudo systemctl enable httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /
usr/lib/systemd/system/httpd.service.
```

> Repeat steps 1-5 for the other two Web Servers.

> Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If you see the same files - it means NFS is mounted correctly. 

6. Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs.
   
 > Before mounting, try to backup the log file, as mounting would replace everything in there and that could be catastrophic

`$ sudo mv /var/log/httpd /var/log/httpdyke`

> **Gotcha!** Remember to change the user and group of /var/log/httpd from nobody to root chmod 700 (This was changed to nobody by the NFS, remember it's already in sync with the Web server?). If not you'll run into a bit of snag where apache isn't starting up

`$ sudo mount -t -o nfs rw,nosuid 10.154.0.6:/mnt/logs`

7. Fork the tooling source code from Darey.io Github Account to your Github account.

`$ git clone https://github.com/darey-io/tooling.git`


8. Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html

```
$ cd /var/www/html
$ ls

admin_tooling.php  img        README.md     tooling_stylesheets.css
create_user.php    index.php  register.php
functions.php      login.php  style.css
```

9. Update the website’s configuration to connect to the database (in functions.php file), then apply the tooling-db.sql script.

`$ mysql -h 34.82.54.113 -u webaccess -p to
oling < tooling-db.sql
Enter password: `

10.  Disable SELinux -  open following config file `sudo vi /etc/sysconfig/selinux `  and set `SELINUX=disabled.`

and run the following commands

```
$ sudo setsebool -P httpd_can_network_connect 1
$ sudo setsebool -P httpd_use_nfs 1
$ sudo setsebool -P httpd_can_network_connect_db 1

```

11. Open the website in your browser http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php and make sure you can login into the website with admin admin.

![](https://github.com/Arafly/DevOps_Tooling_Website_Solution/blob/master/assets/phplogin.PNG)

![](https://github.com/Arafly/DevOps_Tooling_Website_Solution/blob/master/assets/dashboard.png)
