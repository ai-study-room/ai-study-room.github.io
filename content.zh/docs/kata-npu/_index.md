---
weight: 1
bookFlatSection: true
title: "Kata支持NPU"
---

# Kata使用NPU场景

kata因为runtime增加了一层kernel，使其隔离性更好，安全更高，常常用于对隔离要求比较高的场景，如等保。金融行业或者其它对安全要求较高的行业或者领域，解决如规避容器逃逸风险问题。

## Kata简介

kata全称为kata container，是一个开源的，遵循OCI标准的安全容器运行时，采用虚拟化技术，使其使用起来跟容器一样。更多详情参见[kata官网](https://katacontainers.io)

## NPU简介

NPU(Nautral-network Processing Unit)，全称为神经网络处理单元，是一种采用特殊的硬件结构，专用于处理人工智能人物而设计的处理器。

这里特指华为昇腾NPU，更多详情参见[华为官网](https://e.huawei.com/cn/products/computing/ascend)

## Kata支持NPU直通方案

要使主机上的NPU设备能直通到kata容器内部，需要处理两阶段的设备挂载。

一、 将主机上的设备插入到Guest的PCI上

该阶段主要采用将NPU设备切换成vfio设备，然后通过动态插拔插件以热插拔的方式将其到guest内核的PCI中作为guest的常规设备呈现。
将NPU相关驱动打包到guest镜像中，虚拟机启动时默认加载相关驱动，即可在虚拟机内正常使用NPU。

二、 将Guest上NPU设备挂载到容器内部namespace里。

该阶段的工作与将主机上的NPU挂载到常规容器里相同，主要就是在容器启动阶段将NPU设备符以及相关的驱动目录bind mount到对应namespace下。
该部分通过OCI hook实现对应的功能。但由于kata在第一阶段就已经确定了需要加载哪些设备到guest里，guest到容器内部只需将所有NPU都挂载即可。
逻辑上与常规hook存在差异。

要完成上述两阶段的任务，需要完成如下工作：

1. 重新准备guest镜像（包括kernel与rootfs），镜像需默认包含NPU驱动安装。
2. 构建qemu版本使其支持NPU服务器。
3. 改造hook以及kata-agent使其支持将guest中的设备挂载到容器内部。
4. 如果需要将kata作为k8s后端运行，需要提供vfio类型的device plugin。
5. 应用依赖NPU相关的编程接口（CANN），提供CANN的默认基础镜像。
6. 应用需要做相应的改造，以codellama为例提供案例。

## Kata支持NPU直通涉及的组件

* [**ascend-kata-img**](https://github.com/ai-study-room/ascend-kata-img) 主要是用于构建kata guest镜像（包括guest kernel、kata-agent、rootfs、npu驱动等）。项目包括了镜像构建的过程文档相关的过程工具、配置文件等。

* [**kata-containers**](https://github.com/ai-study-room/kata-containers) 主要是kata项目源码，fork自[上游社区](https://github.com/kata-containers/kata-containers)，原则上可以使用上游社区，但是由于kata原生不直接支持NPU，在实际过程中做了一些修改，这些修改还在逐步同步到上游社区过程中。
 
* [**ascend-kata-hook**](https://github.com/ai-study-room/ascend-kata-hook) 主要用于将kata guest kernel中的npu设备以及驱动相关程序自动加载到容器namespace。

* [**vfio-device-plugin**](https://github.com/ai-study-room/vfio-device-plugin) k8s的device plugin，该组件能自动扫描kubelet组件所在的主机上的vfio设备并上报给k8s调度。同时项目中还提供了脚本能将NPU社区切换为vfio设备。

* [**cann**](https://github.com/ai-study-room/cann) 提供华为异构计算框架的基础容器镜像，基于该镜像可以方便构建出kata应用镜像。该镜像结果发布在[https://hub.docker.com/r/slob/cann](https://hub.docker.com/r/slob/cann)

* [**codellama**](https://github.com/ai-study-room/codellama) 经过改造能直接运行在NPU上的meta 开源的代码生成大模型,该项目主要用于测试。

