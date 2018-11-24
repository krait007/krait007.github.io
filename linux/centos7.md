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

```
yum install -y net-tools sysstat ethtool
yum install -y git

```



### Configure CentOS yum repo

```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
cd /etc/yum.repos.d/
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache

#update os system
yum -y update
```



### Install Docker

```
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



### 