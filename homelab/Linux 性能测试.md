# Linux 性能测试

### 1. 检查 cpu、硬盘的温度

>对于服务器，更好的办法是通过 node_exporter 等监控工具采集指标，通过 grafana 等面板展示，这样比手动一个个跑命令要方便快捷非常多！

性能测试也需要注意硬件的温度情况，如果温度过高会导致性能下降，就需要考虑补充散热措施了。

Linux 下使用 sensors 命令查看硬盘温度，需要先安装 lm-sensors 软件包：

```shell
# debian 系安装指令： apt install lm-sensors
# arch 系安装指令： pacman -S lm_sensors
$ sensors
# 固态硬盘温度
nvme-pci-0300
Adapter: PCI adapter
Composite:    +52.9°C  (low  = -273.1°C, high = +89.8°C)
                       (crit = +94.8°C)
Sensor 1:     +52.9°C  (low  = -273.1°C, high = +65261.8°C)
Sensor 2:     +40.9°C  (low  = -273.1°C, high = +65261.8°C)

# cpu socket 接口旁边传感器的温度，这个温度意义不大，除非它很高
acpitz-acpi-0
Adapter: ACPI interface
temp1:        +27.8°C  (crit = +105.0°C)

# intel cpu 温度
## 如果是 amdcpu，这个一般是叫 k10temp-pci-00c3
coretemp-isa-0000
Adapter: ISA adapter
Package id 0:  +41.0°C  (high = +80.0°C, crit = +100.0°C)
Core 0:        +39.0°C  (high = +80.0°C, crit = +100.0°C)
Core 4:        +36.0°C  (high = +80.0°C, crit = +100.0°C)
Core 8:        +37.0°C  (high = +80.0°C, crit = +100.0°C)
Core 12:       +36.0°C  (high = +80.0°C, crit = +100.0°C)
Core 16:       +39.0°C  (high = +80.0°C, crit = +100.0°C)
Core 20:       +35.0°C  (high = +80.0°C, crit = +100.0°C)
Core 24:       +36.0°C  (high = +80.0°C, crit = +100.0°C)
Core 25:       +36.0°C  (high = +80.0°C, crit = +100.0°C)
Core 26:       +36.0°C  (high = +80.0°C, crit = +100.0°C)
Core 27:       +36.0°C  (high = +80.0°C, crit = +100.0°C)
Core 28:       +37.0°C  (high = +80.0°C, crit = +100.0°C)
Core 29:       +37.0°C  (high = +80.0°C, crit = +100.0°C)
Core 30:       +37.0°C  (high = +80.0°C, crit = +100.0°C)
Core 31:       +37.0°C  (high = +80.0°C, crit = +100.0°C)

# 也是固态硬盘温度
nvme-pci-0200
Adapter: PCI adapter
Composite:    +42.9°C  (low  = -273.1°C, high = +89.8°C)
                       (crit = +94.8°C)
Sensor 1:     +42.9°C  (low  = -273.1°C, high = +65261.8°C)
Sensor 2:     +31.9°C  (low  = -273.1°C, high = +65261.8°C)
```

其中两块 nvme 硬盘的温度不太好分辨，可以通过如下命令查出所有块设备与 PCI 地址的对应关系：

```shell
cd /sys/block
for i in nvme*; do
    echo "$i is `cat $i/device/address`"
done
```

我的输出为：

```
nvme0n1 is 0000:02:00.0
nvme1n1 is 0000:03:00.0
```

再通过 `lsblk` 查看设备的挂载情况：

```
$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    1    0B  0 disk 
nvme0n1     259:0    0  1.8T  0 disk 
├─nvme0n1p1 259:1    0 1000M  0 part /efi
└─nvme0n1p2 259:2    0  1.8T  0 part /var/log
                                     /var/cache
                                     /home
                                     /swap
                                     /
nvme1n1     259:3    0  1.9T  0 disk 
├─nvme1n1p1 259:4    0  100M  0 part 
├─nvme1n1p2 259:5    0   16M  0 part 
├─nvme1n1p3 259:6    0  1.9T  0 part 
└─nvme1n1p4 259:7    0  697M  0 part 
```

结合上述信息可知，温度高的 `nvme-pci-0300` 对应的是 `/dev/nvme0n1`，它正是我的系统盘，正在使用中，所以温度高也正常。

而 `nvme-pci-0200` 对应的是 `/dev/nvme1n1`，它是我的 Windows 系统盘，目前没有在使用，所以温度低。

### 其他硬件相关指令

```shell
# 内存信息
sudo dmidecode -t memory

# cpu 信息
sudo dmidecode -t processor
# 这个也可以查看 cpu 信息
lscpu

# 硬盘信息（机械、固态都可以）
sudo smartctl -a /dev/nvme0n1

# 网卡详细信息
sudo ethtool eno1
```

### 使用 sysbench 进行性能测试

cpu 内存测试：

```shell
# 测试 CPU 性能
sysbench cpu --threads=16 --time=30 run

# 测试内存性能
sysbench memory --threads=16 --time=30 run

# 测试文件系统性能
## 在 /tmp 目录下测试系统盘性能
cd /tmp

## 首先准备测试文件
sysbench fileio --file-total-size=10G prepare
# 执行顺序读写测试
sysbench fileio --file-test-mode=seqwr --time=300 --max-requests=0 run
## 然后进行测试
sysbench fileio cleanup
```

### 检测 USB 设备的协议版本

```shell
lsusb -vvv |grep -i -B5 -A2 bcdUSB
```

lsusb 列出信息中的 bcdUSB 即为协议版本，如下所示：

```
Bus 003 Device 002: ID 0e8d:0608 MediaTek Inc. Wireless_Device
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.10
  bDeviceClass          239 Miscellaneous Device
  bDeviceSubClass         2 
--
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            9 Hub
  bDeviceSubClass         0 
--
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               3.10
  bDeviceClass            9 Hub
  bDeviceSubClass         0 
```

可以通过查看块设备的符号链接来确认 USB/NVMe 设备对应的 PCI 地址：

```shell
ls -la /sys/dev/block/ | grep -Ev "loop|dm"
```

至于速度测试，可以直接使用上一节提到的 `sysbench fileio` 来做。


## 硬盘性能问题排查

如果是机械硬盘，速度慢的可能原因有：

1. 使用的是 Windows 中常用的 NTFS/FAT 文件系统，长期使用可能会导致磁盘碎片严重，需要进行磁盘碎片整理
2. 磁盘使用不当，坏道较多，需要进行磁盘检查

关于 Linux 中磁盘碎片的问题，详见 [如何在 Linux 中整理磁盘碎片 ](https://linux.cn/article-6295-1.html)

## 命令行 GPU 监控

- [nvtop](https://github.com/Syllo/nvtop): GPUs process monitoring for AMD, Intel and NVIDIA 

