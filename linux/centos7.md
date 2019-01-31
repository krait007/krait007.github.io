# Centos 7.5 Install Guide

### Prepare Usb  or CDROM install media 

```
1. download CentOs iso from https://www.centos.org/download/
2. download tool: Universal USB Installer 
3. use "Universal" make usb centos installer:  select 	 "centos installer"
```



### Install CentOs OS

```
1. restart from usb
2. setup centos  
   Localization:  
      Date&time select:  asia / shanghai
      Languages Support add:  简体中文
   SOFTWARE: "Minimal Install"
   ...
   
```



### Install tools

```bash
yum install -y net-tools sysstat ethtool telnet
yum install -y git

```



### Configure CentOS yum repo

```bash
yum install -y wget
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
cd /etc/yum.repos.d/
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache

#update os system
yum -y update
```



### Install GNOME Desktop

```bash
#get group install list
yum grouplist
yum groupinstall "GNOME Desktop" "Graphical Administration Tools"

```



### Configure Firewall

```
firewall-cmd --state
firewall-cmd --get-service
firewall-cmd --reload

#add port
firewall-cmd --permanent --zone=public --add-port=8000/tcp
firewall-cmd --permanent --zone=public --add-port=8080-8081/tcp
firewall-cmd --zone=public --add-port=8080-8081/tcp 

#add service
firewall-cmd --zone=public --add-service=https 
firewall-cmd --permanent --zone=public --add-service=https
```



### Install Docker

```bash
#update system pkgs
yum -y update

#run docker setup script
curl -sSL https://get.docker.com/ | sh
yum install -y docker-selinux

systemctl start docker.service
#enable auto start docker service
systemctl enable docker.service

#add user to docker group
usermod -aG docker username

#chage docker images and container volume
mkdir /newmnt_volume
cp -rf /var/lib/docker/* /newmnt_volume/
mv /var/lib/docker /var/lib/docker_backup
ln -s /new/mnt_volume /var/lib/docker


#No route to host
#By default, firewalld will block intercontainer networking on the same docker host
firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 4 -i docker0 -j ACCEPT
firewall-cmd --reload
systemctl restart docker

```



### Install Kvm

```bash
yum install kvm libvirt python-virtinst qemu-kvm virt-viewer tunctl bridge-utils avahi dmidecode qemu-kvm-tools virt-manager qemu-img virt-install net-tools libguestfs-tools -y
```



### Install golang

```
# download binary package: 
wget https://dl.google.com/go/go1.11.2.linux-amd64.tar.gz

tar xvf go1.11.2.linux-amd64.tar.gz
mv go /usr/local/go1.11.2
cd /usr/local
ln -s go1.11.2 go

echo 'export GOHOME=/usr/local/go' > /etc/profile.d/golang.sh
echo 'export PATH=$PATH:$GOHOME/bin' >> /etc/profile.d/golang.sh

#set GOPATH
#for example

cd $HOME 
mkdir bin pkg src
vi ~/.bash_profile
export GOPATH=$HOME/go

#download golang pkg
go get golang.org/x/tour
...

```



Install gcc

```
yum install -y gcc gcc-c++ gdb
```



### Config Ignore Notebook Lid Switch

```
/etc/systemd/logind.conf
HandleLidSwitch=ignore

systemctl restart systemd-logind

#manul enter standby mode
systemctl suspend
```



### 