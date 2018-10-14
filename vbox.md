# VirtualBox 常用命令

## 虚拟机管理
### 查看有哪些虚拟机
```
VBoxManage list vms
```

### 查看虚拟的详细信息
```
VBoxManage list vms --long
```

### 查看运行着的虚拟机
```
VBoxManage list runningvms
```

### 开启虚拟机在后台运行
```
VBoxManage startvm backup -type headless
```

### 开启虚拟机并开启远程桌面连接的支持
```
VBoxManage startvm <vm_name> -type vrdp
```

### 改变虚拟机的远程连接端口,用于多个vbox虚拟机同时运行
```
VBoxManage controlvm <vm_name> vrdpprot <ports>
```

### 关闭虚拟机
```
VBoxManage controlvm <vm_name> acpipowerbutton
```

### 强制关闭虚拟机
```
VBoxManage controlvm <vm_name> poweroff
```

### 删除虚拟机
```
VBoxManage unregistervm     <uuid|vmname> [--delete]
```
