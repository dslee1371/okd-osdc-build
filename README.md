# OKD Install

## cluster name , domain name
cluster1.dslee.lab

## Host
l1-01-base1.cluster1.dslee.lab, 10.0.0.1 , 192.168.1.59
l1-02-lb1.cluster1.dslee.lab, 10.0.0.2
l1-03-nfs1.cluster1.dslee.lab, 10.0.0.3
l1-31-master1.cluster1.dslee.lab, 10.0.0.31
l1-32-master2.cluster1.dslee.lab, 10.0.0.32
l1-33-master3.cluster1.dslee.lab, 10.0.0.33
l1-41-infra1.cluster1.dslee.lab, 10.0.0.41
l1-42-infra2.cluster2.dslee.lab, 10.0.0.42
l1-71-node1.cluster1.dslee.lab, 10.0.0.71
l1-72-node2.cluster1.dslee.lab, 10.0.0.72

## VM
l1-01-base1 - DNS/Musquarade/HA Proxy - 130g/200g/200g (OS/Docker/Yum), 4c 4G,/d/
l1-02-lb1 - Load Balancer - 130g, 2c 2G,/e/
l1-31-master1 - Master - 130g/200g, 4c 16G,/d/
l1-32-master2 - Master - 130g/200g,4c 16G,/e/
l1-33-master3 - Master - 130g/200g,4c 16G,/e/
l1-41-infra - Infra - 130g/200g,4c 4G,/d/
l1-42-infra - Infra - 130g/200g,4c 5G,/e/
l1-71-node1 - Compute - 130g/200g,4c 8G,/d/
l1-72-node2 - Compute - 130g/200g,4c 8G,/e/

## Install
- hyper-v add disk 200g : docker image, 200g : yum package 
- hostname : l1-01-base1.cluster1.dslee.lab
- Partitioning
```
fdisk /dev/sdb     #docker images
n
p
t #8e
w

pvcreate /dev/sdb1
vgcreate vg-docker /dev/sdb1
lvcreate -n lv-docker -l 100%FREE vg-docker

mkfs.xfs /dev/mapper/vg--docker-lv--docker

mkfs.xfs -f -ssize=4k /dev/mapper/vg--docker-lv--docker

fsck -y /dev/mapper/vg--docker-lv--docker

mkdir /volume-dockerimages
mount /dev/mapper/vg--docker-lv--docker /volume-dockerimages

vi /etc/fstab
#/dev/mapper/registry--vg-registry--lv         /volumes          ext4    defaults        0 0
/dev/mapper/registry--vg-registry--lv         /volumes          xfs    defaults        0 0
/dev/mapper/vg--docker-lv--docker /volume-dockerimages    /volume-dockerimages    xfs    defaults    0 0

fdisk /dev/sdc    #yum packages

pvcreate /dev/sdc1
vgcreate vg-yum /dev/sdc1
lvcreate -n lv-yum -l 100%FREE vg-yum


mkfs.xfs /dev/mapper/vg--yum-lv--yum


mkfs.xfs -f -ssize=4k /dev/mapper/vg--yum-lv--yum


fsck -y /dev/mapper/vg--yum-lv--yum


mkdir /var/www/html/repos  # httpd tools 설치하는 이후에 수행
mount /dev/mapper/vg--yum-lv--yum /var/www/html/repos


vi /etc/fstab
#/dev/mapper/registry--vg-registry--lv         /volumes          ext4    defaults        0 0
/dev/mapper/registry--vg-registry--lv         /volumes          xfs    defaults        0 0
/dev/mapper/vg--yum-lv--yum /volume-yums    /var/www/html/repos    xfs    defaults    0 0
```

- Remove Partition
```
umount /dev/mapper/vg--docker-lv--docker
lvremove /dev/vg-docker/lv-docker
vgremove vg-docker
pvremove /dev/sdb1
fdisk /dev/sdb
d /w
```

- Docker Storage install
```
#!/bin/sh
. ./00_env.sh
echo -e "[S]================================================================================"
for hostdomain in `cat ${OCP_HOSTNAME}`
do
echo -e "-----------------------------------------------------------------------------------"
echo -e "@@@[DOCKER SETTING] ==> ${hostdomain}"
ssh -tt root@$hostdomain " \
yum -y install docker; \
echo 'DEVS=${DOCKER_DEVS}' > /etc/sysconfig/docker-storage-setup; \
echo 'VG=docker-vg' >> /etc/sysconfig/docker-storage-setup; \
echo 'WIPE_SIGNATURES=true' >> /etc/sysconfig/docker-storage-setup; \
docker-storage-setup; \
systemctl enable docker; \
systemctl restart docker; "

echo -e "-----------------------------------------------------------------------------------"
done
echo -e "================================================================================[E]"
```
- Set up and Configure Yum Repositories  
```
sudo yum install httpd
sudo yum install createrepo
sudo yum install yum-utils
sudo mkdir –p /var/www/html/repos/{base,centosplus,extras,updates}
sudo reposync -g -l -d -m --repoid=base --newest-only --download-metadata --download_path=/var/www/html/repos/
sudo reposync -g -l -d -m --repoid=centosplus --newest-only --download-metadata --download_path=/var/www/html/repos/
sudo reposync -g -l -d -m --repoid=extras --newest-only --download-metadata --download_path=/var/www/html/repos/
sudo reposync -g -l -d -m --repoid=updates --newest-only --download-metadata --download_path=/var/www/html/repos/
sudo createrepo /var/www/html

–g – lets you remove or uninstall packages on CentOS that fail a GPG check
–l – yum plugin support
–d – lets you delete local packages that no longer exist in the repository
–m – lets you download comps.xml files, useful for bundling groups of packages by function
––repoid – specify repository ID
––newest-only – only download the latest package version, helps manage the size of the repository
––download-metadata – download non-default metadata
––download-path – specifies the location to save the packages
```

- Repo Client  
```
mv /etc/yum.repos.d/*.repo /tmp/
sudo nano /etc/yum.repos.d/remote.repo

[remote]
name=RHEL Apache
baseurl=http://192.168.1.10
enabled=1
gpgcheck=0

```

- Firewall-cmd 
```
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --zone=internal --change-interface=eth1
firewall-cmd --list-all-zones
firewall-cmd --get-active-zone
firewall-cmd --permanent --add-service=http --zone=internal
firewall-cmd --reload

# node to node
firewall-cmd --permanent --zone=internal --add-port=4789/udp

# node to master
firewall-cmd --permanent --zone=internal --add-port=4789/udp --add-port=8443/tcp

# master to node
firewall-cmd --permanent --zone=internal --add-port=4789/udp --add-port=10250/tcp --add-port=10010/tcp


# master to master
firewall-cmd --permanent --zone=internal --add-port=2049/tcp --add-port=2049/udp --add-port=2379/tcp --add-port=2380/tcp --add-port=4789/udp

# external to master
firewall-cmd --permanent --zone=public --add-port=8443/tcp --add-port=8444/tcp

# all
firewall-cmd --permanent --zone=internal --add-port=4789/udp
firewall-cmd --permanent --zone=internal --add-port=4789/udp --add-port=10250/tcp --add-port=10010/tcp
firewall-cmd --reload

# master
firewall-cmd --permanent --zone=internal --add-port=4789/udp --add-port=8443/tcp
firewall-cmd --permanent --zone=internal --add-port=2049/tcp --add-port=2049/udp --add-port=2379/tcp --add-port=2380/tcp --add-port=4789/udp
firewall-cmd --permanent --zone=public --add-port=8443/tcp --add-port=8444/tcp
firewall-cmd --reload

```

- Configure Internal IP
- Configure Name Service
- Configure Masquerade

- Install package
```
yum -y install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct 
```

- epel package
```
yum -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo
yum -y --enablerepo=epel install ansible pyOpenSSL

cd ~
git clone https://github.com/openshift/openshift-ansible
cd openshift-ansible
git checkout release-3.11

```

- build nfs server
```

[root@localhost ~]# fdisk -l

Disk /dev/sda: 136.4 GB, 136365211648 bytes, 266338304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: dos
Disk identifier: 0x000d87dd

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   266338303   132119552   8e  Linux LVM

Disk /dev/sdb: 322.1 GB, 322122547200 bytes, 629145600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/mapper/centos-root: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/mapper/centos-home: 79.4 GB, 79448506368 bytes, 155172864 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes

[root@localhost ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x1bbb5e3d.

The device presents a logical sector size that is smaller than
the physical sector size. Aligning to a physical sector (or optimal
I/O) size boundary is recommended, or performance may be impacted.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-629145599, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-629145599, default 629145599):
Using default value 629145599
Partition 1 of type Linux and of size 300 GiB is set

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8
Changed type of partition 'Linux' to 'AIX'

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8e
Changed type of partition 'AIX' to 'Linux LVM'

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.


[root@localhost ~]# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
[root@localhost ~]# vgcreate nfs-vg /dev/sdb1
  Volume group "nfs-vg" successfully created
[root@localhost ~]# lvcreate -n nfs-lv -l 100%FREE nfs-vg
  Logical volume "nfs-lv" created.


[root@localhost ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/centos/swap
  LV Name                swap
  VG Name                centos
  LV UUID                oyWz6K-QE0y-aAMJ-692v-xVtw-NLLF-fSZFHV
  LV Write Access        read/write
  LV Creation host, time localhost, 2020-06-03 22:49:25 +0900
  LV Status              available
  # open                 2
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/centos/home
  LV Name                home
  VG Name                centos
  LV UUID                cbs4Ks-fBj8-uQd0-tv61-mVZg-rs1H-cjhNPM
  LV Write Access        read/write
  LV Creation host, time localhost, 2020-06-03 22:49:26 +0900
  LV Status              available
  # open                 1
  LV Size                73.99 GiB
  Current LE             18942
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

  --- Logical volume ---
  LV Path                /dev/centos/root
  LV Name                root
  VG Name                centos
  LV UUID                9kxHcF-ClDK-XBda-iyGd-K9qV-cS7o-mFqzOG
  LV Write Access        read/write
  LV Creation host, time localhost, 2020-06-03 22:49:26 +0900
  LV Status              available
  # open                 1
  LV Size                50.00 GiB
  Current LE             12800
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0

  --- Logical volume ---
  LV Path                /dev/nfs-vg/nfs-lv
  LV Name                nfs-lv
  VG Name                nfs-vg
  LV UUID                78Wjzy-Uee5-O9zd-pTSj-rZL4-HDeE-XWwRpn
  LV Write Access        read/write
  LV Creation host, time l1-03-nfs1.cluster1.dslee.lab, 2020-09-02 23:23:18 +0900
  LV Status              available
  # open                 0
  LV Size                <300.00 GiB
  Current LE             76799
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:3

[root@localhost ~]# mkfs.xfs /dev/nfs-vg/nfs-lv
meta-data=/dev/nfs-vg/nfs-lv     isize=512    agcount=4, agsize=19660544 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=78642176, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=38399, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@localhost ~]# fsck -y /dev/nfs-vg/nfs-lv
fsck from util-linux 2.23.2
/sbin/fsck.xfs: XFS file system.


[root@localhost ~]# mkdir /volumes
[root@localhost ~]# mount /dev/nfs-vg/nfs-lv /volumes
[root@localhost ~]# vi /etc/fstab

[root@localhost volumes]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Wed Jun  3 09:49:27 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=b8a4849b-8210-46c5-85f9-34b3371e0a92 /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
/dev/nfs-vg/nfs-lv      /volumes        xfs     defaults        0 0


```


## Verify Check 
```
# dns query
l1-03-nfs1.cluster1.dslee.lab
l1-31-master1.cluster1.dslee.lab 
l1-32-master2.cluster1.dslee.lab 
l1-33-master3.cluster1.dslee.lab 
l1-41-infra1.cluster1.dslee.lab
l1-42-infra2.cluster1.dslee.lab
l1-71-node1.cluster1.dslee.lab
l1-72-node2.cluster1.dslee.lab

```



## Referrence Documents
- https://github.com/okd-community-install/installcentos 
