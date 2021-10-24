# PREPARE NFS SERVER

To start I will we using my AWS account to create an EC2 instance with Red-Hat as the OS. This will become my **NFS Server**.

Use link:
[Creating Red Hat instance in AWS](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/AWS_ReHat_Instnace.md)

Following the same steps I did for a previous project [WEB-SOLUTION-WITH-WORDPRESS](https://github.com/hectorproko/WEB-SOLUTION-WITH-WORDPRESS/blob/main/Steps_WebSolutionWordpress.md) I will create 3 **Elastic Block Store** Volumes to **attach** to the **NFS Server**. Unlike the previous project this we'll format the disks as **xfs** instead of **ext4**. We need to end up with 3 **Logical Volumes** 
* **lv-apps** to be mounted on **mnt/apps** - To be used by webservers
* **lv-logs** to be mounted on **mnt/logs** - To be used by webserver logs
* **lv-opt**  to be mounted on **/mnt/opt** - To be used by Jenkins server later project

After partitioning the **EBS** Volumes
``` bash
[ec2-user@ip-172-31-93-202 ~]$ sudo lvmdiskscan
  /dev/xvda2 [     <10.00 GiB]
  /dev/xvdf1 [     <10.00 GiB]
  /dev/xvdg1 [     <10.00 GiB]
  /dev/xvdh1 [     <10.00 GiB]
  0 disks
  4 partitions #3 newly create + already existing one xvda2
  0 LVM physical volume whole disks
  0 LVM physical volumes

[ec2-user@ip-172-31-93-202 ~]$
```

Creating **Physical Volumes**
``` groovy
[ec2-user@ip-172-31-88-22 ~]$ sudo pvcreate /dev/xvdf1
  Physical volume "/dev/xvdf1" successfully created.
[ec2-user@ip-172-31-88-22 ~]$ sudo pvcreate /dev/xvdg1
  Physical volume "/dev/xvdg1" successfully created.
[ec2-user@ip-172-31-88-22 ~]$ sudo pvcreate /dev/xvdh1
  Physical volume "/dev/xvdh1" successfully created.
[ec2-user@ip-172-31-88-22 ~]$
```

Checking newly created **Physical Volumes**
``` bash
[ec2-user@ip-172-31-93-202 ~]$ sudo lvmdiskscan
  /dev/xvda2 [     <10.00 GiB]
  /dev/xvdf1 [     <10.00 GiB] LVM physical volume #here
  /dev/xvdg1 [     <10.00 GiB] LVM physical volume #here
  /dev/xvdh1 [     <10.00 GiB] LVM physical volume #here
  0 disks
  1 partition
  0 LVM physical volume whole disks
  3 LVM physical volumes #Total

[ec2-user@ip-172-31-93-202 ~]$
```

Turning each **Physical Volume** into a **Volume Group**
```groovy
[ec2-user@ip-172-31-88-22 ~]$ sudo vgcreate vg-apps /dev/xvdf1
  Volume group "vg-apps" successfully created
[ec2-user@ip-172-31-88-22 ~]$ sudo vgcreate vg-logs /dev/xvdg1
  Volume group "vg-logs" successfully created
[ec2-user@ip-172-31-88-22 ~]$ sudo vgcreate vg-opt /dev/xvdh1
  Volume group "vg-opt" successfully created
[ec2-user@ip-172-31-88-22 ~]$
```
Checking newly created **Volume Groups** with **vgs**
``` bash
[ec2-user@ip-172-31-93-202 ~]$ sudo vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  vg-apps   1   0   0 wz--n- <10.00g <10.00g 
  vg-logs   1   0   0 wz--n- <10.00g <10.00g
  vg-opt    1   0   0 wz--n- <10.00g <10.00g
[ec2-user@ip-172-31-93-202 ~]$
```
Turning each **Volume Group** to a **Logical Volume**
```bash
#-l 100%VG, ensures 100% of avaiable space of VG is used
[ec2-user@ip-172-31-93-202 ~]$ sudo lvcreate -l 100%VG -n lv-apps vg-apps
  Logical volume "lv-apps" created.
[ec2-user@ip-172-31-93-202 ~]$ sudo lvcreate -l 100%VG -n lv-logs vg-logs
  Logical volume "lv-logs" created.
[ec2-user@ip-172-31-93-202 ~]$ sudo lvcreate -l 100%VG -n lv-opt vg-opt
  Logical volume "lv-opt" created.

[ec2-user@ip-172-31-93-202 ~]$
```
Checking newly created **Logical Volumes** with **lvs**
```bash
[ec2-user@ip-172-31-93-202 ~]$ sudo lvs
  LV      VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv-apps vg-apps -wi-a----- <10.00g
  lv-logs vg-logs -wi-a----- <10.00g
  lv-opt  vg-opt  -wi-a----- <10.00g
[ec2-user@ip-172-31-93-202 ~]$
```

Creating **mounting points** (directories)
``` bash
[ec2-user@ip-172-31-93-202 ~]$ sudo mkdir /mnt/apps
[ec2-user@ip-172-31-93-202 ~]$ sudo mkdir /mnt/logs
[ec2-user@ip-172-31-93-202 ~]$ sudo mkdir /mnt/opt
[ec2-user@ip-172-31-93-202 ~]$ ls /mnt/
apps  logs  opt # < newly created
[ec2-user@ip-172-31-93-202 ~]$
```

We can take a broader look with **lsblk**
```bash
[ec2-user@ip-172-31-93-202 ~]$ sudo lsblk
NAME                  MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda                  202:0    0  10G  0 disk
├─xvda1               202:1    0   1M  0 part
└─xvda2               202:2    0  10G  0 part /
xvdf                  202:80   0  10G  0 disk #Device/Volume
└─xvdf1               202:81   0  10G  0 part #Partition
  └─vg--apps-lv--apps 253:0    0  10G  0 lvm  #VolumeGroup-LogicalVolume
xvdg                  202:96   0  10G  0 disk
└─xvdg1               202:97   0  10G  0 part
  └─vg--logs-lv--logs 253:1    0  10G  0 lvm
xvdh                  202:112  0  10G  0 disk
└─xvdh1               202:113  0  10G  0 part
  └─vg--opt-lv--opt   253:2    0  10G  0 lvm
[ec2-user@ip-172-31-93-202 ~]$
```
Formatting the **Logical Volumes** to **xfs**
```bash
[ec2-user@ip-172-31-93-202 ~]$ sudo mkfs.xfs /dev/vg-apps/lv-apps
[ec2-user@ip-172-31-93-202 ~]$ sudo mkfs.xfs /dev/vg-logs/lv-logs
[ec2-user@ip-172-31-93-202 ~]$ sudo mkfs.xfs /dev/vg-opt/lv-opt
[ec2-user@ip-172-31-93-202 ~]$
```

Manual Mount
``` bash
sudo mount /dev/vg-apps/lv-apps /mnt/apps/
sudo mount /dev/vg-logs/lv-logs /mnt/logs/
sudo mount /dev/vg-opt/lv-opt /mnt/opt/
```
Adding **mounts** to **fstab** after getting **Logical Volume** UUID using **blkid**
```bash 
UUID=Suffs0-E50z-6bwx-kCFk-5cfN-ecnI-3s0Xm4 /mnt/apps             xfs     defaults        0 0
UUID=BAPXhu-XZ0I-S0Go-0uAT-EcFj-KJzX-6etnCA /mnt/logs             xfs     defaults        0 0
UUID=LuEjkp-3IbF-9C3n-2TuM-21Qe-fR95-Frwfar /mnt/opt              xfs     defaults        0 0
```
Use **sudo mount -a** check for errors <br>
Lets set up permissions that will allow our Web servers to read, write and execute files on NFS:

Installing **NFS server**, configure to start on reboot and make sure it is up and running
```bash
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service #start on boot
sudo systemctl status nfs-server.service 
```
We set up permission that will allow our Web servers to read, write and execute files on NFS
```bash
sudo chown -R nobody: /mnt/apps #changes user and group to nobody, a way to tell the kernel that any user can read and write access to the file
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
sudo systemctl restart nfs-server.service
```
```bash
[ec2-user@ip-172-31-88-22 ~]$ ls -l /mnt/
total 0
drwxrwxrwx. 2 nobody nobody 6 Sep 22 00:18 apps
drwxrwxrwx. 2 nobody nobody 6 Sep 22 00:18 logs
drwxrwxrwx. 2 nobody nobody 6 Sep 22 00:18 opt
[ec2-user@ip-172-31-88-22 ~]$
```
For simplicity, in this project all Instances (tiers) are inside the same **subnet**
Now I need to check which **subnet** my **NFS Server** is in

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/Devops-Tooling-Website-Solution/main/images/subnet.png)
 <br>
 ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/Devops-Tooling-Website-Solution/main/images/subnet2.png)
 <br>

 As we can see **IPv4 CIDR** is **172.31.80.0/20**

Configuring access to NFS for clients within the same subnet 
``` perl
[ec2-user@ip-172-31-88-22 ~]$ sudo vi /etc/exports #edit this file
[ec2-user@ip-172-31-88-22 ~]$ cat /etc/exports 
/mnt/apps 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash) #added
/mnt/logs 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash) #added
/mnt/opt 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash) #added

[ec2-user@ip-172-31-88-22 ~]$ sudo exportfs -arv #used to maintain the current table of exported file systems for NFS
exporting 172.31.80.0/20:/mnt/opt
exporting 172.31.80.0/20:/mnt/logs
exporting 172.31.80.0/20:/mnt/apps

[ec2-user@ip-172-31-88-22 ~]$
```

Now we make sure the following ports are open to hosts from the same subnet **172.31.80.0/20** <br>
Use link:
[Opening Ports in AWS](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/OpenPortAWS.md)

 ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/Devops-Tooling-Website-Solution/main/images/ports.png)
 <br>

 # CONFIGURE THE DATABASE SERVER
The following takes place in **NFS Server** <br>
```bash
sudo yum install mysql-server -y #Installation 
sudo mysql_secure_installation #additional setup

sudo systemctl start mysqld #starting it

sudo mysql #to access MySQL
```
Creating a database called **tooling** and user **webaccess** with all priviledges from **subnet 172.31.80.0/20**
``` sql
mysql> CREATE DATABASE `tooling`;
Query OK, 1 row affected (0.01 sec)

mysql> CREATE USER `webaccess`@`172.31.80.0/20` IDENTIFIED WITH mysql_native_password BY 'Passw0rd!'
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL ON tooling.* TO 'webaccess'@'172.31.80.0/20';
Query OK, 0 rows affected (0.00 sec)
```
# PREPARE THE WEB SERVERS
We need to make sure that our Web Servers can serve the same content from shared storage solutions, in this case NFS Server and MySQL database. This means we will be able to add new ones or remove them whenever we need, and the integrity of the data (in the database and on NFS) will be preserved making the webservers **stateless**

we will utilize NFS and mount previously created Logical Volume lv-apps to the folder where Apache stores files to be served to the users (/var/www).
