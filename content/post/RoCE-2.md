+++
date = '2025-09-04T11:00:00+08:00'
draft = 'true'
title = 'RoCE带宽时延测试'
+++
## 测试目的

评估 RoCE网卡接口在 RDMA 场景下的性能表现，主要性能指标包括：

- 带宽（Bandwidth）：评估网卡最大吞吐能力
- 时延（Latency）：衡量数据包往返所需时间

## 相关名词概念解释

1. RoCE:RDMA over Converged Ethernet,RDMA技术在以太网上的实现
2. RDMA:RemoteDirect Memory Access，远程内存直接访问
   1. 面对高性能计算、大数据分析等IO高并发、低时延应用，现有TCP/IP软硬件架构不能满足应用的需求，这主要体现在传统的TCP/IP网络通信是通过内核发送消息，这种通信方式存在很高的数据移动和数据复制的开销。RDMA就是为了解决网络传输中服务器端数据处理的延迟而产生的。如下图，RDMA技术能直接通过网络接口访问内存数据，无需操作系统内核的介入。这允许高吞吐、低延迟的网络通信，尤其适合在大规模并行计算机集群中使用
   2. ![img](https://iqeubg8au73.feishu.cn/space/api/box/stream/download/asynccode/?code=MTc4YTBiYTEzZmY5MzA3MDkzZmFiNjY4ZjhiZjlmNmNfRnA1ZDI5RVVaa3FIaUxPbEc4TFRuQjlETFVlY1lla3pfVG9rZW46V1N1dmJRZFpZb3JlSGJ4TDl0dWNDQWNXbmJnXzE3NTcwMzkxMjE6MTc1NzA0MjcyMV9WNA)
3. 读操作RDMA Read：客户端从服务端读取数据，服务端被动响应，延迟较低
4. 写操作RDMA Write：客户端将数据直接写入服务端内存，常用于大数据吞吐
5. 发送操作RDMA Send：传统网络传输

## 测试前准备

本次测试为单机，选择2个卡打流

1. 网卡信息，NPU机器除了业务口的网卡，其他的只能通过hccn_tool操作
   ```bash
   for i in {0..15}; do    echo "=== Device $i ==="    hccn_tool -i $i -link -g 2>/dev/null    hccn_tool -i $i -ip -g 2>/dev/null    echo done # 一共有16张HCCN网卡
   ```
2. 网卡节点信息
   1.   执行下面这个命令，system name tlv相同的两个设备是同轨，不同的是异轨
   2. ```bash
      #!/bin/bash for i in {0..15}; do    echo "Interface $i:"    hccn_tool -i $i -lldp -g | grep "System Name TLV" -A 1
      echo done # 输出如下：
      Interface 0: System Name TLV        SH-YS2-M203-J12U28-H3CS9825-G0-04001
      Interface 1: System Name TLV        SH-YS2-M203-J12U33-H3CS9825-G0-03001
      Interface 2: System Name TLV        SH-YS2-M203-J12U43-H3CS9825-G0-01001
      # .....省略
      Interface 8: System Name TLV        SH-YS2-M203-J12U33-H3CS9825-G0-03001
      Interface 9: System Name TLV        SH-YS2-M203-J12U28-H3CS9825-G0-04001
      ```
3. 网卡IP地址
   ```bash
   # hccn_tool -i id -ip -g
   root@infra-gpu-npu-011:~# hccn_tool -i 0 -ip -g ipaddr:10.250.45.69 netmask:255.255.255.0
   root@infra-gpu-npu-011:~# hccn_tool -i 1 -ip -g ipaddr:10.250.30.30 netmask:255.255.255.0
   root@infra-gpu-npu-011:~# hccn_tool -i 8 -ip -g ipaddr:10.250.45.28 netmask:255.255.255.0
   ```

## 测试步骤

1. 在接收端执行**hccn_tool -i id -roce_test type [-d %s] [-s %d] [-a] [-b] [-f %d] [-D %d] [-l %d] [-m %d] [-ib %d] [-p %d] [-q %d] [-Q %d] [-t %d] [-u %d] [-V] [-x %d][-n %d] [-tclass %d] -tcp**命令，进入等待连接状态
2. 在发送端执行**hccn_tool -i id -roce_test type [-d %s] [-s %d] [-a] [-b] [-f %d] [-D %d] [-l %d] [-m %d] [-ib %d] [-p %d] [-q %d] [-Q %d] [-t %d] [-u %d] [-V] [-x %d][-n %d] [-tclass %d] address ipaddr -tcp**命令

**注意：需要接收端先发出命令，发送端后发出命令，否则会连接失败，上述命令【】中参数为可选，其余为必选**

### 同轨发送操作带宽测试

```Bash
# 接收端 device 9
root@infra-gpu-npu-011: hccn_tool -i 9 -roce_test ib_send_bw -s 2097152 -n
 1000 -tclass  132 -tcp
Dsmi get perftest status end. (status=1)
Dsmi start roce perftest end. (out=1)
Dsmi get perftest status end. (status=2)
roce_report:
 new post send flow is not supported, falling back to ibv_post_send
--------------------------------------------------------------------------------------------------------------------

************************************
* Waiting for client to connect... *
************************************
--------------------------------------------------------------------------------------------------------------------
...   # 等待发送端
# 发送端 device 0
root@infra-gpu-npu-011:~# hccn_tool -i 0 -roce_test ib_send_bw -s 2097152 -n
 1000 -tclass 132 address 10.250.45.28 -tcp
Dsmi get perftest status end. (status=1)
Dsmi start roce perftest end. (out=1)
Dsmi get perftest status end. (status=1)
roce_report:
 new post send flow is not supported, falling back to ibv_post_send
--------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : hns_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
--------------------------------------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x004a PSN 0xe03644
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:250:45:69
 remote address: LID 0000 QPN 0x0012 PSN 0x5785c2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:250:30:26
--------------------------------------------------------------------------------------------------------------------
 #bytes       #iterations        BW peak[MB/sec]            BW average[MB/sec]          MsgRate[Mpps]
 2097152      1000               23370.21                   23365.53                    0.011683
--------------------------------------------------------------------------------------------------------------------
# 接收端
--------------------------------------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : hns_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 RX depth        : 512
 CQ Moderation   : 1
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
--------------------------------------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0012 PSN 0x5785c2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:250:30:26
 remote address: LID 0000 QPN 0x004a PSN 0xe03644
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:250:45:69
--------------------------------------------------------------------------------------------------------------------
 #bytes       #iterations        BW peak[MB/sec]            BW average[MB/sec]          MsgRate[Mpps]
 2097152      1000               0.00                       23394.57                    0.011697
--------------------------------------------------------------------------------------------------------------------
```

- 成功则打印带宽值。
- 异常可通过以下命令主动终止-i对应device侧OS的roce_test进程。
  - **hccn_tool -i 0 -roce_test reset**

若需更改发送操作为 读/写 ，更改带宽测试为时延测试，只需更改roce_test;若需更改同轨为异轨，只需更改设备id和接收端IP。具体步骤与上述类似，此处不再赘述。

附上四份表格以供参考

|                    | 同轨读操作带宽测试 | 同轨写操作带宽测试 | 同轨发送操作带宽测试 | 异轨读操作带宽测试 |          |          |          |          |
| ------------------ | ------------------ | ------------------ | -------------------- | ------------------ | -------- | -------- | -------- | -------- |
|                    | 发送端             | 接收端             | 发送端               | 接收端             | 发送端   | 接收端   | 发送端   | 接收端   |
| bytes              | 1048576            | 1048576            | 1048576              | 1048576            | 1048576  | 1048576  | 1048576  | 1048576  |
| iterations         | 1000               | 1000               | 1000                 | 1000               | 1000     | 1000     | 1000     | 1000     |
| BW peak[MB/sec]    | 23375.37           | 23375.37           | 23375.31             | 23375.31           | 23364.33 | 0        | 23364.42 | 23364.42 |
| BW average[MB/sec] | 23366.23           | 23366.23           | 23368.05             | 23368.53           | 23363.58 | 23394.90 | 23364.15 | 23364.15 |
| MsgRate[Mpps]      | 0.023366           | 0.023366           | 0.023369             | 0.023369           | 0.023364 | 0.023395 | 0.023364 | 0.023364 |

|                    | 异轨读操作带宽测试 | 异轨写操作带宽测试 | 异轨发送操作带宽测试 |          |          |          |
| ------------------ | ------------------ | ------------------ | -------------------- | -------- | -------- | -------- |
|                    | 发送端             | 接收端             | 发送端               | 接收端   | 发送端   | 接收端   |
| bytes              | 1048576            | 1048576            | 1048576              | 1048576  | 1048576  | 1048576  |
| iterations         | 1000               | 1000               | 1000                 | 1000     | 1000     | 1000     |
| BW peak[MB/sec]    | 23364.42           | 23364.42           | 23364.37             | 23364.37 | 23353.34 | 0        |
| BW average[MB/sec] | 23364.15           | 23364.15           | 23363.87             | 23363.87 | 23350.1  | 23390.38 |
| MsgRate[Mpps]      | 0.023364           | 0.023364           | 0.023364             | 0.023364 | 0.02335  | 0.02339  |

BW average【MB/sec】基本稳定在23360MB/sec

|                        | 同轨写操作时延测试 | 同轨发送操作时延测试 | 同轨读操作时延测试 |        |        |        |
| ---------------------- | ------------------ | -------------------- | ------------------ | ------ | ------ | ------ |
|                        | 发送端             | 接收端               | 发送端             | 接收端 | 发送端 | 接收端 |
| t_min[usec]            | 46.08              | 46.08                | 46.60              | 48.75  | 48.50  |        |
| t_max[usec]            | 131.19             | 130.94               | 115.95             | 226.82 | 109.10 |        |
| t_typical[usec]        | 46.23              | 46.23                | 46.81              | 48.99  | 48.72  |        |
| t_avg[usec]            | 46.51              | 46.53                | 47.10              | 49.33  | 48.84  |        |
| t_stdev[usec]          | 2.79               | 2.83                 | 2.28               | 4.08   | 1.88   |        |
| 99% percentile[usec]   | 54.20              | 60.73                | 56.94              | 56.29  | 49.42  |        |
| 99.9% percentile[usec] | 131.11             | 130.94               | 115.95             | 226.82 | 109.15 |        |

|                        | 异轨写操作时延测试 | 异轨发送操作时延测试 | 异轨读操作时延测试 |        |        |        |
| ---------------------- | ------------------ | -------------------- | ------------------ | ------ | ------ | ------ |
|                        | 发送端             | 接收端               | 发送端             | 接收端 | 发送端 | 接收端 |
| t_min[usec]            | 50.29              | 50.29                | 50.85              | 50.85  | 56.6   |        |
| t_max[usec]            | 162.23             | 161.95               | 288.82             | 227.79 | 137.6  |        |
| t_typical[usec]        | 50.46              | 50.45                | 51.09              | 51.11  | 56.8   |        |
| t_avg[usec]            | 50.92              | 50.99                | 51.86              | 51.69  | 56.91  |        |
| t_stdev[usec]          | 4.27               | 4.79                 | 7.41               | 4.68   | 1.08   |        |
| 99% percentile[usec]   | 68.4               | 69.72                | 84.39              | 66.39  | 57.36  |        |
| 99.9% percentile[usec] | 162.23             | 161.95               | 288.82             | 227.79 | 137.6  |        |

可以看到同轨时延相比于异轨时延明显降低了，同轨带宽略高于异轨

### 打印信息分析

- BW average[MB/sec]：带宽平均值
- MsgRate (Mpps) = 每秒发送的消息条数,百万数据包/s,带宽 = MsgRate × 消息大小 × 8
- t_stdev[usec]：标准差，3.42 微秒，说明波动比较小
- percentile[usec]：传输时间的百分位数，99.9% percentile[usec]：144.28,99.9% 的请求延迟小于144.28微秒

### 必选参数解析

- -i id：指定设备ID
- -roce_test type：
  - ib_read_bw：读操作带宽测试
  - ib_write_bw：写操作带宽测试
  - ib_send_bw：发送操作带宽测试
  - ib_read_lat：读操作时延测试
  - ib_write_lat：写操作时延测试
  - ib_send_lat：发送操作时延测试
  - reset：复位指令，终止对应device OS的所有roce_test进程。**Ctrl + C 不能终止**
- address ipaddr：接收端网卡设备IP地址，用于给发送端建立连接
- -tcp：指定建链方式为tcp建链

### 非必选参数解析

- -s：指定msg包的大小，取值范围：1~874736000。单位：B。不加该参数默认值为65536
  ```bash
  root@infra-gpu-npu-011:~# hccn_tool -i 8 -roce_test ib_send_bw -s 1048576 -n 1000 -tclass  132 -tcp  #bytes       #iterations        BW peak[MB/sec]            BW average[MB/sec]          MsgRate[Mpps] 1048576      1000               0.00                       23392.64                    0.023393
  root@infra-gpu-npu-011:~# hccn_tool -i 8 -roce_test ib_send_bw -s 2097152 -n 1000 -tclass  132 -tcp  #bytes       #iterations        BW peak[MB/sec]            BW average[MB/sec]          MsgRate[Mpps] 2097152      1000               0.00                       23394.57                    0.011697
  ```

-s:1048576, 每条消息大小 1MB->2097152, 每条消息大小 2MB; MsgRate[Mpps]:0.23393->0.11697

 也就是说：

- 2MB 消息：消息大，但每秒能处理的条数少 → MsgRate 低
- 1MB 消息：消息小，处理条数多 → MsgRate 高。但是带宽未必更高，因为虽然消息数多了，单条消息小了，总吞吐量可能差不多，甚至更低


- -n：指定迭代次数

