---
title: 环境准备
weight: 1
---

# NPU相关的服务器

根据需求准备服务器多台并安装相应的操作系统以及NPU驱动
详细参见[华为官网](https://www.hiascend.com/document/detail/zh/quick-installation/23.0.0/quickinstg_train/900_A2_PoDc/quickinstg_900A2_PoDc__0001.html),注意服务器上安装驱动与固件即可。开发套件、CANN、pytorch、算子包等均不需安装。
安装完成后，确保npu能正常管理，通过命令行执行npu-smi命令验证：

```shell
[root@myserver]# npu-smi info

+------------------------------------------------------------------------------------------------+
| npu-smi 23.0.rc3                 Version: 23.0.rc3                                             |
+---------------------------+---------------+----------------------------------------------------+
| NPU   Name                | Health        | Power(W)    Temp(C)           Hugepages-Usage(page)|
| Chip                      | Bus-Id        | AICore(%)   Memory-Usage(MB)  HBM-Usage(MB)        |
+===========================+===============+====================================================+
| 1     910B3               | OK            | 91.7        50                0    / 0             |
| 0                         | 0000:01:00.0  | 0           0    / 0          4314 / 65536         |
+===========================+===============+====================================================+
| 2     910B3               | OK            | 93.0        48                0    / 0             |
| 0                         | 0000:C2:00.0  | 0           0    / 0          4305 / 65536         |
+===========================+===============+====================================================+
| 3     910B3               | OK            | 92.5        49                0    / 0             |
| 0                         | 0000:02:00.0  | 0           0    / 0          5591 / 65536         |
+===========================+===============+====================================================+
| 4     910B3               | OK            | 93.6        50                0    / 0             |
| 0                         | 0000:81:00.0  | 0           0    / 0          4159 / 65536         |
+===========================+===============+====================================================+
| 5     910B3               | OK            | 97.1        50                0    / 0             |
| 0                         | 0000:41:00.0  | 0           0    / 0          4159 / 65536         |
+===========================+===============+====================================================+
| 6     910B3               | OK            | 99.6        51                0    / 0             |
| 0                         | 0000:82:00.0  | 0           0    / 0          4318 / 65536         |
+===========================+===============+====================================================+
| 7     910B3               | OK            | 98.3        49                0    / 0             |
| 0                         | 0000:42:00.0  | 0           0    / 0          4317 / 65536         |
+===========================+===============+====================================================+
+---------------------------+---------------+----------------------------------------------------+
| NPU     Chip              | Process id    | Process name             | Process memory(MB)      |
+===========================+===============+====================================================+
| No running processes found in NPU 1                                                            |
+===========================+===============+====================================================+
| No running processes found in NPU 2                                                            |
+===========================+===============+====================================================+
| No running processes found in NPU 3                                                            |
+===========================+===============+====================================================+
| No running processes found in NPU 4                                                            |
+===========================+===============+====================================================+
| No running processes found in NPU 5                                                            |
+===========================+===============+====================================================+
| No running processes found in NPU 6                                                            |
+===========================+===============+====================================================+
| No running processes found in NPU 7                                                            |
+===========================+===============+====================================================+

```

## BIOS 开启SMMU

重启服务器，并进入BIOS设置,修改如下配置

```shell
BIOS 页面 --> MISC Config --> Support Smmu
BIOS 页面 --> MISC Config --> Smmu Work Around
```

内核配置启动参数, 内核配置文件 grub.cfg 添加参数 iommu=pt intel_iommu=on，并重启节点。


## NPU设备切VFIO

进入服务器命令行。

1. 加载vfio-pci驱动

```shell
[root@myserver]# modprobe vfio-pci
```

检查驱动模块是否成功加载

```shell
[root@myserver]# lsmod | grep -i vfio
vfio_pci               61440  1
vfio_mdev              16384  0
mdev                   24576  1 vfio_mdev
vfio_virqfd            16384  1 vfio_pci
vfio_iommu_type1       40960  0
vfio                   36864  7 vfio_mdev,vfio_iommu_type1,vfio_pci
```

其中显示vfio_pci则证明加载成功。

2. 将NPU设备驱动切换为vfio设备类型

通过npu-smi info命令找到各NPU卡Bus-Id，格式为0000:c2:00.0(后续均以NPU 2卡为例，见上图)，通过以下两种方式切换：

* 方法一，通过vfio-device-plugin项目提供的脚本切换:

下载脚本
[https://github.com/ai-study-room/vfio-device-plugin/blob/main/vfio-pci-bind.sh](https://github.com/ai-study-room/vfio-device-plugin/blob/main/vfio-pci-bind.sh)
可以页面点击下载，或者执行命令下载：

```shell
wget https://raw.githubusercontent.com/ai-study-room/vfio-device-plugin/main/vfio-pci-bind.sh
```


执行脚本切换：
```shell
vfio-pci-bind.sh 0000:c2:00.0
```

* 方法二，通过命令切换

```shell
export BDF="0000:c2:00.0"
echo $BDF > /sys/bus/pci/drivers/devdrv_device_driver/unbind
echo vfio-pci > /sys/bus/pci/devices/$BDF/driver_override
echo $BDF > /sys/bus/pci/drivers_probe
```

3. 检查切换是否成功

切换成功后，在/dev/vfio/ 目录下将能查看对应的设备

```shell
[root@myserver]# ls /dev/vfio
70  vfio
```

其中70就是切换成功后的设备，同时可以查看设备的驱动类型是否为vfio-pci。

```shell
[root@myserver]# lspci -vs 0000:c2:00.0
......
Kernel driver in use: vfio-pci
	Kernel modules: drv_vascend, dbl_runenv_config, drv_devmm_host, drv_devmm_host_agent, ascend_event_sched_host, drv_devmng_host, drv_pcie_hdc_host, ascend_trs_sec_eh_agent, ascend_trs_sub_stars, ascend_queue, drv_davinci_intf_host, ts_agent, ascend_trs_pm_adapt, dbl_dev_identity, dbl_algorithm, drv_soft_fault, ascend_soc_platform, drv_dvpp_cmdlist, drv_pcie_host, drv_pcie_vnic_host, ascend_trs_shrid, drv_dp_proc_mng_host, ascend_xsmem, drv_virtmng_host
```

其中Kernel driver in use显示为vfio-pci则证明切换成功！


