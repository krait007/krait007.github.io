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







