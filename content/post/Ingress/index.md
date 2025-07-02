+++
date = '2025-07-02T10:37:07+08:00'
draft = true
title = 'Ingress'
tag = 'Ingress'
+++

## Ingress
随着时间的积累原来不懂的东西现在逐渐看清了  
Ingress：网关入口  
集群中Service的统一网管入口  
当Ingress安装完成并启动后，会为其创建一个Service，并设置一个NodePort，因为Ingress要处理所有的外部请求，故必须将Ingress暴露出去，此时访问集群人一台机器的指定端口就可以映射到Ingress上，然后通过在Ingress配置相应的规则，就可以实现对Service的统一网关入口

