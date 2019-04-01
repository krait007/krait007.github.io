# centos7 network bond config

### backup if cfg-xxxx config files

```
cd /etc/sysconfig/network-scripts/
mkdir backup
cp ifcfg-* ./backup
```

### use nmcli command config bond

```bash
nmcli connection add type bond ifname bond0 mode 6
nmcli connection add type bond-slave ifname enoxxxxx master bond0
nmcli connection add type bond-slave ifname enoxxxxx master bond0
```

**bond的mode如下：**

- balance-rr (0) –轮询模式，负载均衡（bond默认的模式）
- active-backup (1) –主备模式（常用）
- balance-xor (2)
- broadcast (3）
- 802.3ad (4) –聚合模式
- balance-tlb (5)
- balance-alb (6)



# ubuntu network bond config

### install fenslave

```
sudo dpkg -l | grep fenslave
sudo apt-get install ifenslave -y
```

### 加载绑定内核模块

```bash
sudo modprobe bonding 
sudo lsmod | grep bonding

sudo vi /etc/modules
bonding       #添加的内容，使该模块开机启动             

```

### 配置网络接口

```bash
sudo vi /etc/network/interfaces
-----------------------------------
auto eno3
iface eno3 inet manual
bond-master bond0

auto eno4
iface eno4 inet manual
bond-master bond0

auto bond0
iface bond0 inet static
onboot yes
address 192.168.1.250
gateway 192.168.1.1
netmask 255.255.255.0
bond-mode 6
bond-miimon 100
bond-slaves  eno3  eno4

---------------------------------
#重启网络服务
sudo /etc/init.d/networking restart
```



