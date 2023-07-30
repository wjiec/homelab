Hardware Passthrough
---------------------------------

开启硬件直通之前，需要确定机器的 CPU 及主板是否支持「VT-d」技术。查询 CPU 是否支持 VT-d ，Intel 用户可到 https://www.intel.cn 搜索自己 CPU 型号，在「规格」中查阅。AMD 用户可到 https://www.amd.com 进行查询。



### 启用 IOMMU 功能

编辑 `/etc/default/grub`文件，在文档中找到 `GRUB_CMDLINE_LINUX_DEFAULT` 项。对于 Intel 用户，将其修改为：

```plain
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

对于  AMD 用户，将改行修改为：

```plain
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

最后需要更新 GRUB：

```shell
update-grub
```



### 加载内核模块

执行以下命令加载对应的内核模块并更新系统内核，并重启 PVE

```shell
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
echo "vfio_virqfd" >> /etc/modules

update-initramfs -k all -u
reboot
```



### 验证是否开启成功

在终端中执行以下命令：

```shell
$ dmesg | grep iommu
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.15.102-1-pve root=/dev/mapper/pve-root ro quiet intel_iommu=on iommu=pt
[    0.321311] Kernel command line: BOOT_IMAGE=/boot/vmlinuz-5.15.102-1-pve root=/dev/mapper/pve-root ro quiet intel_iommu=on iommu=pt
[    1.294747] iommu: Default domain type: Passthrough (set via kernel command line)
[    1.334536] pci 0000:00:00.0: Adding to iommu group 0
[    1.334567] pci 0000:00:01.0: Adding to iommu group 1
[    1.334597] pci 0000:00:02.0: Adding to iommu group 2
[    1.334630] pci 0000:00:03.0: Adding to iommu group 3
[    1.334662] pci 0000:00:05.0: Adding to iommu group 4
[    1.334692] pci 0000:00:05.1: Adding to iommu group 5
...

$ find /sys/kernel/iommu_groups/ -type l
/sys/kernel/iommu_groups/55/devices/0000:80:00.0
/sys/kernel/iommu_groups/83/devices/0000:ff:14.3
/sys/kernel/iommu_groups/17/devices/0000:00:1f.2
/sys/kernel/iommu_groups/17/devices/0000:00:1f.0
/sys/kernel/iommu_groups/45/devices/0000:7f:16.2
/sys/kernel/iommu_groups/73/devices/0000:ff:10.0
/sys/kernel/iommu_groups/73/devices/0000:ff:10.7
/sys/kernel/iommu_groups/73/devices/0000:ff:10.5
/sys/kernel/iommu_groups/73/devices/0000:ff:10.1
/sys/kernel/iommu_groups/73/devices/0000:ff:10.6
...
```

检查命令结果是否与以上内容匹配，如果匹配则说明开启成功。



### 显卡直通

**一般地，不推荐将显卡直通给特定虚拟机，有些命令执行不好会造成不可挽回的损失。建议将直通显卡的虚拟机不要勾选「开机自启动」，以便出现问题还可以用命令行救回。**

依据自己显卡型号，在命令行输入以下命令：

```shell
# 直通 AMD 显卡，请使用下面命令

echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf
 
# 直通 Nvidia 显卡，请使用下面命令
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist nvidiafb" >> /etc/modprobe.d/blacklist.conf
 
# 直通 Intel 核显，请使用下面命令
echo "blacklist snd_hda_intel" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist snd_hda_codec_hdmi" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist i915" >> /etc/modprobe.d/blacklist.conf 
```



### 常见问题

1. 如果经过以上步骤还不能开启 IOMMU，则需要修改 `/etc/kernel/cmdline` 文件，编辑这个文件并根据自己的 CPU 新增一行 `intel_iommu=on` 配置。



### 参考文档

- https://never666.uk/1631/
- https://foxi.buduanwang.vip/virtualization/pve/561.html/
- https://zhuanlan.zhihu.com/p/535623506
- https://forum.level1techs.com/t/proxmox-no-iommu-detected-please-activate-it/179947/2