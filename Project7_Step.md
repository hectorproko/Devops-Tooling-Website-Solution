# Prepare NFS Server

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
Use **sudo mount -a** check for errors

Installing **NFS server**, configure to start on reboot and make sure it is up and running
```bash
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service #start on boot
sudo systemctl status nfs-server.service 
```
For simplicity, in this project all Instances (tiers) are inside the same **subnet**

Now I need to check which **subnet** my **NFS Server** is in

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/WEB-SOLUTION-WITH-WORDPRESS/main/Images/port3306.png)
 <br>