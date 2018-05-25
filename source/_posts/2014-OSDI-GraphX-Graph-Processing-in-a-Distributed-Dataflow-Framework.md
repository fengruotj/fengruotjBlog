title: GraphX Graph Processing in a Distributed Dataflow Framework
author: Master.TJ
tags:
  - 图计算
categories:
  - 研究生论文研读
date: 2018-05-25 17:34:00
---
## 解决的问题：

![upload successful](\blog\images\pasted-26.png)
虽然现有的专用图系统能够实现广泛的系统优化，但也是有代价的。 图只是较大的分析过程的一部分，通常将非结构化的图形和表格式数据组合在一起。 因此，分析流水线（例如图11）被迫组成多个系统，这增加了复杂性并导致不必要的数据移动和重复。 此外，为了追求性能，图形处理系统通常会放弃容错，以支持快照恢复。 最后，作为专门的图处理系统，图处理框架通常不能享受分布式数据流框架的广泛支持。

相比之下，通用分布式数据流框架（例如Map-Reduce [10]，Spark [39]，Dryad [15]）暴露丰富的数据流操作符（例如map，reduce，group-by，join） 用于分析非结构化和表格化数据，并被广泛采用。 但是，直接使用数据流操作符来实现迭代图算法可能具有挑战性，往往需要复杂连接的多个阶段。 此外，分布式数据流框架中定义的通用连接和聚合策略不利用迭代图算法中的常见模式和结构，因此错过了重要的优化机会。

为了支持这个论点，我们引入了GraphX，一个嵌入到Spark [39]分布式数据流系统中的高效的图形处理框架。

它首先对每个Triplet三元组视图的每个元素进行一个MapFunction操作，得到一个消息。这个消息包括，属性数据以及一个目标顶点的ID。就是说你要将这个消息发送到哪个顶点上去。Message Combiners通过累加这些消息，然后将累加结构更新到相应顶点的属性数据。GraphX同这样一个GAS的思想，进行图形迭代计算。这个思想很像MapReduce的思想。通过这样的操作，不停的进行迭代计算。最终图形中每个顶点的属性数据收敛不再变化。整个图形算法最终完成。


## 3.3  GraphX operators

**GraphX Pregel abstraction**:

它的主要思想是通过构造出这样一个Triplet 三元组视图这样的一个结构。这个结构是通过将顶点RDD和边RDD进行Join操作得到

这样一个三元组信息。为什么要得到这样一个Triplet 三元组视图这样的一个结构。最主要的原因是因为这个三元组视图包括源顶点属性、目标顶点属性、以及边属性。这样我们就可以通过一些类似于专业图处理系统的GAS计算模型进行 类似MpaReduce思想的图形计算。
     它首先对每个Triplet三元组视图的每个元素进行一个MapFunction操作，得到一个消息。这个消息包括，属性数据以及一个目标顶点的ID。就是说你要将这个消息发送到哪个顶点上去。Message Combiners通过累加这些消息，然后将累加结构更新到相应顶点的属性数据。GraphX同这样一个GAS的思想，进行图形迭代计算。这个思想很像MapReduce的思想。通过这样的操作，不停的进行迭代计算。最终图形中每个顶点的属性数据收敛不再变化。整个图形算法最终完成。


## GraphX优化
### Vertex Mirroring
由于GraphX的顶点和边属性集合分区都是独立的，然而Join操作需要数据移动。GraphX则通过将顶点属性通过网络传输到边分区上。由于两个原因，这种方法大大减少了通信。 首先，现实世界的图通常比顶点具有更多数量级的边。 其次，单个顶点在同一个分区中可能有许多边，从而实现顶点属性的大量重用。

### Multicast Join(多路广播)
虽然所有顶点被发送到每个边缘分区的广播连接将确保连接发生在边缘分区上，但是由于大多数分区只需要一小部分顶点来完成连接，所以它仍然是低效的。 因此，GraphX引入了多播连接，其中每个顶点属性仅发送到包含相邻边的边缘分区。

![upload successful](\blog\images\pasted-32.png)

### Partial Materialization （部分实体化）
当顶点属性改变时，顶点复制被急切地执行，但是，在边缘分区上的本地join没有实现，以避免重复。 相反，镜像顶点属性存储在每个边界分区上的散列映射中，并在构建三元组时引用。

### Incremental View Maintenacne
当下一次访问三元组视图时，只有已更改的顶点被重新路由到其边缘分割连接点，并且未更改的顶点的局部Mirror被重用。然后被更改的顶点更新相应的Triplets View视图。

![upload successful](\blog\images\pasted-31.png)

### mrTriplets 优化
#### 4.3.1 Filtered Index Scanning
mrTriplets操作的第一阶段就是通过扫描triplets三元组视图，然后应用用户定义的map 函数应用到每个triplets三元组中。然而随着迭代算法的进行，工作集合被收缩，大多数顶点已经收敛。Map function只需要操作那些活跃顶点的triplets三元组视图。直接顺序扫描所有的索引将会变得很浪费。

为了解决这个问题，GraphX提出了Index Scanning对于triplets三元组视图。也就是说应用程序通过使用subgraph运算符来限制图来表示当前活跃顶点。活跃顶点通过route table (查询活跃顶点属于相应的edge partitions)被推送edge partitions，它可以用来使用源顶点id上的CSR索引来快速查找到相应的edge（4.1节）。
#### 4.3.2 Automatic Join Elimination
问题：对于triplets三元组视图的某些operator访问来说，可能某些顶点属性没有访问，或则根本一个都没有使用。那么对于triplets三元组视图来说，会造成一部分资源浪费，因为riplets三元组视图被构造的时候所有的source vertex属性和distinction vertex属性都被join构造到triplets视图中。

GraphX使用JVM字节码分析器在运行时检查用户定义的函数，并确定是否引用源或目标顶点属性。如果只引用一个属性，并且三元组视图尚未实现，则GraphX自动重写生成的查询计划 三联视图从三联加入到双向联接。 如果没有引用顶点属性，则GraphX完全消除连接。 这种修改是可能的，因为三元组视图遵循Spark中RDD的惰性语义。 如果用户从不访问三元组视图，则永远不会实现。 因此，对mrTriplets的调用能够重写生成三元组视图的相关部分所需的联接。

### GraphX 容错
Graph实现容错主要是通过丢失分区的剩余副本或者重新计算它们来恢复