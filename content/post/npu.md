+++
date = '2025-08-31T11:00:00+08:00'
draft = 'true'
title = '昇腾NPU基础知识'

+++

### 相关名词概念

- cpu: 中央处理器,通用计算中心，负责系统控制，逻辑判断和复杂任务调度
- gpu: 图形处理器，专攻大规模并行计算，最初为图形渲染设计，现已成为通用并行计算主力
- npu: 神经网络处理器，专为深度学习设计的加速器，优化矩阵乘加等AI运算
- Altas 800: 华为推出的一系列AI服务器产品，包括训练服务器和推理服务器。昇腾是华为的AI处理器系列，Altas是基于昇腾处理器构建的AI硬件产品系列
- H200,H100,H20: 英伟达推出的AI芯片，H20可以说是专为中国市场设计
- ECC: Error-Correcting Code,错误校验与纠正，一种数据容错技术，用于内存，硬盘，缓存等存储或数据传输场景
- MCU: Micro Control Unit,微控制单元
- 算力切分:将物理服务器，集群或芯片等算力资源，按照一定规则或需求分割正多个独立的，可分配的算力单元，以便更高效的满足不同任务对算力的差异化需求
- HCCN:Huawei Collective Communication Network,华为集合通信网络，专为昇腾AI集群设计的高性能集合通信技术与协议栈，解决多卡，多服务器间的高效数据交互问题，支撑大模型训练，分布式推理等需要大规模算力协同的场景
  - 依赖昇腾专用通信硬件：Altas 800训练服务器通常搭载支持Pcle4.0/5.0,RocE的网卡或内置通信模块

### npu-smi: 华为开发的NPU系统管理工具，主要用于收集设备信息，查看健康状态，配置设备参数等

- npu-smi info

```bash
npu-smi info
# 打印信息说明
Chip:芯片id
Device:芯片编号
Health:芯片的健康状态
Power:芯片功率
Temp:芯片温度
AICore:AICore占用率
Memory-Usage:内存占比

npu-smi info -h | --help   # 显示信息查询命令的帮助信息。

npu-smi info -l   # 查询所有npu设备，NPU ID即为后面的-i id

npu-smi info -m   # 查询所有芯片映射关系

npu-smi info -t borad -i id   # 查询指定NPU的详细信息
# 打印信息说明
Model:产品型号
Manufacturer:生产厂家
Software Version:NPU驱动版本
Firmware Version:NPU固件版本
Compatibility:NPU驱动和NPU固件兼容性
Chip Count:芯片个数

npu-smi info -t borad -i id -c chip_id   # 查询指定芯片信息
npu-smi info -t common -i id   # 查询所有芯片常用信息
npu-smi info -t flash -i id   # 查询所有芯片的闪存信息，同理指定芯片后面加上-c chip_id, flash换成memory既是查询内存信息，换成temp就是温度，power就是功率，不再赘述
# flash memory:闪存,非易失性存储器，没有电源供应下能长期保存数据
npu-smi info -t usages -i id   # 查询所有芯片统计信息
npu-smi info -t ssh-enable -i id   # ssh使能:设备的ssh远程管理权限，true or false


npu-smi info watch   # 查询所有设备上所有芯片的监测数据
npu-smi info watch -i id   # 查询指定设备上所有芯片的监测数据
npu-smi info watch -i id -c chip_id -d delay_second -s watch_type   # delay_seconds 每轮查询延迟时长，单位为秒。watch_type 检测类型，p代表功率，t代表温度等等
```

- npu-smi set

```bash
 npu-smi set -h | --help   # 显示配置命令的帮助信息
  
 npu-smi set -t collet-log -i id   # 收集MCU的日志，收集的日志放在/run/mcu_log目录下
 
 npu-smi set -t reset -i id   # 复位标卡，用于重置指定ID的昇腾NPU设备
 
 npu-smi set -t mcu-monitor -i id -c chip_id -d value   # 设置MCU监测状态, 0-false 1-true
 
 npu-smi set -t disk-power -i id -c chip_id -d vale   # 设置机械硬盘上下电, 0-下电,切断机械硬盘电源，使其停止工作；1-上电，接通电源，进入工作状态
 
 npu-smi set -t ssh-enable -i id -d value   # 配置所有芯片的ssh使能状态，配置所有芯片ssh使能状态后，需要用户手动复位标卡生效配置，配置ECC使能状态ecc-enable同理
 
 npu-smi set -t license -i id -f "license"   # 配置用户的证书。证书只能配置一次，否则会配置失败
 
 npu-smi set -t ip -i id -c chip_id -s ipstring   # 设置AI芯片的IP信息，192.168.5.199/255.255.255.0
 
 npu-smi set -t mac-addr -i id -c chip_id -d mac_id -s mac_string   # 设置执行芯片mac地址
 
 npu-smi set -t nve-level -i id -c chip_id -d value   # 设置指定芯片的算力等级,0-low 1-middle 2-high 3-full
 
 npu-smi set -t aicpu-config -i id -c chip_id -d value   # 设置指定芯片的AI CPU数量
```

  

- npu-smi upgrade

```bash
npu-smi upgrade -h | --help   # 显示升级命令的帮助信息

#item有三种类型, mcu bootloader vrd,不同命令支持类型不一样
npu-smi upgrade -q item -i id   # 查询固件的升级状态

npu-smi upgrade -b item -i id   # 查询固件的版本信息

npu-smi upgrade -t item -i id -f file_path   # 启动固件升级

npu-smi upgrade -a item -i id   # 在固件升级完成之后，用该命令生效固件的新版本
```



- npu-smi clear

```bash
npu-smi clear -h | --help   # 显示清楚命令的帮助信息

npu-smi clear -t ecc-info -i id   # 清楚所有芯片ECC错误计数
```

- 算力切分

```bash
npu-smi info -t vnpu-mode   # 查询算力切分模式,有两种模式:docker, VM, docker是容器级切分,vNPU实例作为进程级资源被分配给容器,容器内应用直接调用宿主机上的NPU驱动和运行时; VM是虚拟机级切分, 不同处约等于docker和虚拟机的区别

npu-smi info -t template-info   # 查询算力切分模板信息

npu-smi set -t vnpu-mode -d mode   # 设置算力切分模式, 0-docker 1-VM
```
