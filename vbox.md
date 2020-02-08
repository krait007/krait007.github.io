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

###  命令行创建虚拟机(from vdi)
```
VBoxManage createvm --name "win10_64" --register
VBoxManage modifyvm "win10_64" --memory 8192 --acpi on --boot1 dvd

# none|null|nat|bridged|intnet|hostonly
VBoxManage modifyvm "win10_64" --nic1 nat --bridgeadapter1 eth0

VBoxManage modifyvm "win10_64" --macaddress1 XXXXXXXXXXXX

VBoxManage modifyvm "win10_64" --cpus 2

VBoxManage list ostypes
VBoxManage modifyvm "win10_64" --ostype Windows10_64

#VBoxManage createhd --filename ./win10_64.vdi --size 10000
#VBoxManage storagectl "win10_64" --name "IDE Controller" --add ide

# storagectl and storageattach
VBoxManage storagectl "win10_64" --name "SATA Controller" --add sata --controller IntelAHCI --portcount 1 --bootable on
VBoxManage storageattach "win10_64_init" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium ./win10_64.vdi

VBoxManage storageattach "win10_64" --storagectl "IDE Controller" \
    --port 1 --device 0 --type dvddrive --medium win10_64.iso

#start vm
VBoxManage startvm "win10_64" -type headless
VBoxHeadless --startvm "win10_64" &

```

### vagrant box
```
#package box
vagrant package --base win10_64 --output win10_64.box
vagrant box list
vagrant box add win10_64 win10_64.box
```

### windows
```
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "win10_64"
  config.vm.network "forwarded_port", guest: 3389, host: 3389
  config.vm.communicator = "winrm"
  config.vm.provider "virtualbox" do |v|
    v.memory = 8192
    v.cpus = 2
  end
end

vagrant  up
failed: /usr/lib/ruby/2.5.0/rubygems/core_ext/kernel_require.rb:59:in `require': cannot load such file -- winrm-elevated (LoadError)

gem install winrm
gem install winrm-elevated

```