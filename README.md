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
