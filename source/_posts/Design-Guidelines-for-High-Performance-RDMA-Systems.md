title: ' Design Guidelines for High Performance RDMA Systems'
author: Master.TJ
tags:
  - RDMA
categories:
  - 研究生论文研读
  - RDMA论文研读
date: 2018-05-23 11:11:00
---
## Introduction
随着RDMA技术的发展，RDMA技术越来越被数据中心采用。尽管RDMA的新知名度，使用他们的先进功能以达到最佳效果仍然是软件设计师的挑战。</br>
找到RDMA功能与应用程序之间的有效匹配非常重要。没有一种方法能够适合所有的应用场景，比如说RDMA一个参数的最佳和最差选择在它们的总吞吐量中变化了70倍，并且它们消耗的主机CPU的量变化了3.2倍。在不同的设计中，应用需求的小的变化显著影响RDMA的相对性能。</br>
1. 首先这篇文章的第一个贡献就是它提供了由一组开源的度量工具支持的指导方针，用于评估和优化在使用RDMA NICs时影响端到端吞吐量的最重要的系统因素。对于每个指导方针。作者就如何确定本指南是否相关提供了深入的见解，并讨论了使用NIC的哪些模式最有可能缓解问题。
2. 其次，作者通过第三代RDMA硬件将这些指南应用于微基准和实际系统来评估这些指南的功效。
---

## Background
![upload successful](\blog\images\pasted-1.png)</br>
图显示了RDMA Cluster硬件组件。其中NIC网卡连接的一个或则多个端口连接到PCIe控制器且连接到多核CPU的服务器上。PCIe恐控制器用来接受NIC网卡的PCIe请求到L3 cache中。在现代的Intel 服务器架构中，这个L3 Cache 提供PCIe 事件控制器。