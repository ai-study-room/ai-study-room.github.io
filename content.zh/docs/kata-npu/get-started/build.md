---
title: 相关项目构建
weight: 2
bookToc: false
---

# Kata相关的组件构建

根据前面[概述](../)中描述, 涉及的项目编译包括

 * guest os（包括kernel与rootfs），其中kernel需要打开NPU相关的配置项，rootfs需要将NPU驱动包打入镜像 ，涉及项目[ascend-kata-img](https://github.com/ai-study-room/ascend-kata-img)以及[kata-containers](https://github.com/ai-study-room/kata-containers)。
 * hook，编译kata的prestart hook, 涉及项目[ascend-kata-hook](https://github.com/ai-study-room/ascend-kata-hook)。
 * device-plugin， 主要是k8s需要，涉及项目[vfio-device-plugin](https://github.com/ai-study-room/vfio-device-plugin)。

## 构建Guest内核

1. clone kata项目到本地

```shell
git clone https://github.com/ai-study-room/kata-containers $GOPATH/src/github.com/kata-containers/kata-containers
```

2. 安装所需工具

```shell
apt-get install libclang-dev protobuf flex bison libssl-dev libncurses5-dev libncursesw5-dev
```

3. 增加npu相关配置

```shell
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/packaging/kernel
cat > configs/fragments/arm64/npu.conf << EOF
CONFIG_MODULES=y
CONFIG_MODULE_UNLOAD=y

# CRYPTO_FIPS requires this config when loading modules is enabled.
CONFIG_MODULE_SIG=y

# Support the DMI and PCI iov
CONFIG_DMI=y
CONFIG_PCI_IOV=y
CONFIG_PCI_PRI=y
CONFIG_PCI_PASID=y

CONFIG_VFIO_MDEV=m
CONFIG_VFIO_MDEV_DEVICE=y
EOF
```

去掉nvidia.arm64.conf.in中的CONFIG_IOMMU_SVA。

```shell
vim configs/fragments/gpu/nvidia.arm64.conf.in

#CONFIG_IOMMU_SVA=y
```



4. 构建.config

```shell
./build-kernel.sh -v 5.15.1 -f -g nvidia -d setup
```

其中5.15.1是指内核版本，可以根据实际调整版本，建议使用该版本。
执行成功后会生成文件kata-linux-5.15.1-xx/.config。

5. 打开KVM相关配置

进入kata-linux-5.15.1-xxx目录，执行make menuconfig。

```shell
cd kata-linux-5.15.1-130
make menuconfig
在弹出的对话框中，通过shift+y选择[Virtualization]后，再选中[Kernel-based Virtual Machine(KVM) support]
最后Save 再Exit
```

该修改成功后，将在.config文件中增加KVM相关的配置。

6. 执行内核的编译&安装

```shell
cd ..
./build-kernel.sh -v 5.15.1 -f -g nvidia -d build
./build-kernel.sh -v 5.15.1 -f -g nvidia -d install
```

这个过程需要一些时间，执行成功后将生成内核文件到/usr/share/kata-containers/目录下。

```shell
ls /usr/share/kata-containers/
config-5.15.1-130-nvidia-gpu   vmlinux-5.15.1-130-nvidia-gpu   vmlinux-nvidia-gpu.container   
vmlinuz-5.15.1-130-nvidia-gpu 
```

7. 构建出内核头文件

```shell
cd kata-linux-5.15.1-130 
make deb-pkg
```

执行完后将生成如下文件：

```shell
ls ../*.deb
linux-headers-5.15.1-130_arm64.deb  linux-libc-dev_5.15.1-130_arm64.deb
linux-image-5.15.1-130_arm64.deb
```


## 构建kata-aget

1. 安装musl gcc

```shell
wget https://musl.libc.org/release/musl-1.2.4.tar.gz
tar -xf musl-1.2.4.tar.gz

cd musl-1.2.4
./configure
make
ln -s /usr/local/musl/bin/musl-gcc /usr/bin/$(uname -m)-linux-musl-gcc
```

2. 构建kata-aget


```shell
cd $GOPATH/src/github.com/kata-containers/kata-containers/src/agent
make -e SECCOMP=no -e LIBC=musl kata-agent
make kata-agent.service
```

执行完成后，会在target/xxxxx-linux-musl/release目录下生成二进制包kata-agent，当前目录下生成kata-agent.service与kata-containers.target。


## 构建Hook

1. 下载hook源代码

```shell
git clone https://github.com/ai-study-room/ascend-kata-hook $GOPATH/src/github.com/ai-study-room/ascend-kata-hook
```


2. 编译hook

```shell
cd $GOPATH/src/github.com/ai-study-room/ascend-kata-hook/
go build -buildmode=pie -trimpath -o ./out/ascend-kata-hook ./hook/main.go
```

构建完成后将生成二进制执行文件到out目录。

## 准备rootfs

1. 安装必要工具

```shell
sudo add-apt-repository universe
apt-get install multistrap makedev
```

2. 安装docker用于构建rootfs

```shell
apt install docker.io
```

3. 设置相应的环境变量, 可以使用[env.sh](https://github.com/ai-study-room/ascend-kata-img/blob/main/manifest/scripts/env.sh)代替

```shell
# EXTRA_PKGS 预安装的软件包，这里列出了必须的，还可以根据实际需要增加，如vim之类
export EXTTRA_PKGS="chrony make curl pciutils apt dpkg python3 software-properties-common kmod net-tools udev build-essential"
export USE_DOCKER=true
export ROOTFS_DIR=/root/rootfs-ubuntu-1-15-63  #这里指定最后生成rootfs的路径，根据实际情况设置
# AGENT_SOURCE_BIN指向前面编译的agent二进制文件
export AGENT_SOURCE_BIN=${GOPATH}/src/github.com/kata-containers/kata-containers/src/agent/target/aarch64-unknown-musl/release/kata-agent

```

4. 构建rootfs
```shell
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder
./rootfs.sh ubuntu
```

执行完成后，将在$ROOTFD_DIR下生成基础的rootfs文件系统。

```shell
ls $ROOTFS_DIR
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

5. 复制相应的软件包到rootfs中

复制方法有多种，只需确保文件都完成复制即可。为了方便使用，可以将NPU驱动安装包，内核头文件、kata-agent等文件放置在一个目录下，通过[pkg-copy.sh](https://github.com/ai-study-room/ascend-kata-img/blob/main/manifest/scripts/pkg-copy.sh)脚本完成复制，也可以通过如下命令复制：

```shell
# 设置相关包的路径
MANIFEST_DIR=/root
# 设置kata项目路径
KATA_REPO=$GOPATH/src/github.com/kata-containers/kata-containers
# 复制驱动包到rootfs下
cp $MANIFEST_DIR/Ascend-hdk-910b-npu-driver_23.0.3_linux-aarch64.run $ROOTFS_DIR/root/

# 复制前面编译生成的内核头文件
cp $KATA_REPO/tools/packaging/kernel/linux-libc-dev_5.15.1-130_arm64.deb $ROOTFS_DIR/root/
cp $KATA_REPO/tools/packaging/kernel/linux-headers_5.15.1-130_arm64.deb $ROOTFS_DIR/root/

# 复制驱动加载相关的配置文件及脚本
cp $MANIFEST_DIR/mod_probe.sh $ROOTFS_DIR/usr/local/bin
mkdir -p $ROOTFS_DIR/lib/modules/updates
cp $MANIFEST_DIR/mod.dep $ROOTFS_DIR/lib/modules/updates/

# 复制kata-agent
cp $KATA_REPO/src/agent/kata-agent.service $ROOTFS_DIR/etc/systemd/system/
cp $KATA_REPO/src/agent/kata-containers.target $ROOTFS_DIR/etc/systemd/system/
cp $KATA_REPO/src/agent/target/aarch64-unknown-linux-musl/release/kata-agent $ROOTFS_DIR/usr/bin/
chmod +x $ROOTFS_DIR/usr/bin/kata-agent

# 复制hook
mkdir -p $ROOTFS_DIR/usr/share/oci/hooks/prestart
cp $GOPATH/src/github.com/ai-study-room/ascend-kata-hook/out/ascend-kata-hook $ROOTFS_DIR/usr/share/oci/hooks/prestart/
chmod +x $ROOTFS_DIR/usr/share/oci/hooks/prestart/ascend-kata-hook
``` 

6. 安装NPU驱动到rootfs中

挂载相关文件系统到rootfs中，可以通过[mount.sh](https://github.com/ai-study-room/ascend-kata-img/blob/main/manifest/scripts/mount.sh)完成挂载，也可以通过如下命令挂载：

```shell
mount -t sysfs -o ro none $ROOTFS_DIR/sys
mount -t tmpfs none $ROOTFS_DIR/tmp
mount -t proc -o ro none $ROOTFS_DIR/proc
mount -o bind,ro /dev $ROOTFS_DIR/dev
mount -t devpts none $ROOTFS_DIR/dev/pts
```

chang root 到rootfs根目录

```shell
chroot $ROOTFS_DIR/
```


7. 安装NPU驱动

增加NPU驱动用户，并安装linux头文件。该部分可以通过[init.sh](https://github.com/ai-study-room/ascend-kata-img/blob/main/manifest/scripts/init.sh)完成，也可以痛殴如下命令完成：

```shell
groupadd HwHiAiUser
useradd -g HwHiAiUser -d /home/HwHiAiUser -m HwHiAiUser -s /bin/bash

mkdir -p /var/log  ## 安装驱动时需要使用该目录

dpkg -i /root/linux-headers_5.15.1-130_arm64.deb
dpka -i /root/linux-libc-dev_5.15.1-130_arm64.deb
```

安装驱动包，具体的参数可以通过--help查看意义，建议按如下执行：

```shell
chmod +x /root/Ascend-hdk-910b-npu-driver_23.0.3_linux-aarch64.run
./root/Ascend-hdk-910b-npu-driver_23.0.3_linux-aarch64.run --full --install-for-all
```

安装之后将在/lib/modules/updates目录下。


清理环境退出chroot，该步骤很重要，源文件执行完以后不再使用，没有必要保留。

```shell
rm -rf /root/*
exit
```

umount 所有文件系统，可以通过[umount.sh](https://github.com/ai-study-room/ascend-kata-img/blob/main/manifest/scripts/umount.sh)完成，也可以通过以下命令完成：

```shell
umount $ROOTFS_DIR/sys
umount $ROOTFS_DIR/proc
umount $ROOTFS_DIR/tmp
umount $ROOTFS_DIR/dev/pts
umount $ROOTFS_DIR/dev
```

8. 构建kata镜像

```
cd $KATA_REPO/tools/osbuilder/image-builder/
./image-builder/image_builder.sh $ROOTFS_DIR
```

最终将生成kata镜像kata-containers.img，如下：

```shell
ls -l 
-rw-r--r-- 1 root root 671088640 May 29 15:38 kata-containers.img
```

9. 构建device plugin

```
git clone https://github.com/ai-study-room/vfio-device-plugin $GOPATH/src/github.com/ai-study-room/vfio-device-plugin

cd $GOPATH/src/github.com/ai-study-room/vfio-device-plugin
go mod download
go build -o vfio-device-plugin main.go
```



## 总结

该部分主要有以下产物，用于后续的实际环境安装:

kata-containers.img ---- kata镜像文件
vmlinux-nvidia-gpu.container ---- Guest 内核文件
vfio-device-plugin  ---- k8s相关的device plugin
