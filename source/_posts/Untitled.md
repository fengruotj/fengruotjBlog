title: Fast and Concurrent RDF Queries with RDMA-based Distributed Graph Exploration
author: Master.TJ
tags:
  - RDMA
  - 实时流处理
categories:
  - 研究生论文研读
date: 2018-05-25 17:41:00
---

## RDF图Graph应用场景：
通过对大量且不断增长的RDF数据进行大量查询，RDF图形存储库为并发查询处理提供低延迟和高吞吐量势在必行。
## 现有的工作存在的问题：
然而，先前的系统在大数据集上仍然经历高的查询延迟，并且大多数先前的设计具有较差的资源利用率，使得每个查询被顺序地处理。查询处理集中的依赖于潜在大表的连接操作，这通常会产生巨大的冗余中间数据。 此外，使用关系表triplets来存储三元组可能会限制一般性，使得现有系统难以支持RDF数据的一般图形查询，如可达性分析和社区检测。

## 现有的解决方案：
1. 使用triple 存储和 triple join方法
存在的问题：First,使用三元组存储会过度依赖Join操作，特别是分布式环境下的merge/hash join操作。Second, scan-join操作会产生大量的中间冗余结果。Third, 尽管现有的工作使用redundant six primary SPO4 permutation index 可以加速join操作，但是索引会导致大量的内存开销。
2. 使用Graph store 和 Graph exploration
存在的问题：之前的工作表明，最后一步join相应的子查询结果会造成一个潜在的性能瓶颈。特别是查询那些存在环的语句，或者有很大的中间结果的情况下。‘

### Graph Model And  Graph Indexs
在Wukong中这里有两种不同类型的索引结构。分别是 Predicate Index和Type Index索引。

![upload successful](\blog\images\pasted-35.png)
Wukong提出了预测索引（P-idx）来维护所有使用其特定谓词标记的主体和对象入边和出边。索引顶点本质上充当从谓词到相应的主体或对象的倒排索引。Wukong还提出了一种Type Index索引方便查询一个Subject属于的类型。与先前基于图的方法（使用单独的数据结构管理索引）不同，Wukong将索引作为RDF图的基本部分（顶点和边）处理，同时还考虑了这些索引的分割和存储。 
好处：首先，这使用图探索简化了查询处理，以便图探索可以直接从索引顶点开始。 其次，这使得在多个服务器之间分配索引变得简单而高效。

### Differentiated Graph Partitioning

![upload successful](\blog\images\pasted-38.png)
受到PowerLyra的启发，Wukong采用不同的分区策略算法对于正常顶点和索引顶点来说。每个正常顶点（例如，DS）将被随机分配（即，通过 散列顶点ID）到只有一个机器的所有边缘（邻居的ID）。与正常顶点不同的是，每个索引顶点（例如，takeCourse和Course）将被拆分并复制到多个机器，其边缘链接到同一机器上的正常顶点。 这很自然地将索引和它们的负载分配给每台机器。 

### RDMA-friendly Predicate-based Store

![upload successful](\blog\images\pasted-41.png)
Wukong采用一种基于RDMA-Based的分布式hash表结构存储RDF Graph Data。在这样的结构中，它包含两种不同的索引结构，一种是Type-index索引，存储Subject/Objetc的类型索引。一种是Predicate-Index索引，存储的是谓词的相邻顶点的索引。

### Query Processing Query
1. Basic Query Processing
Wukong利用图探索通过沿着图特别是根据子图的每个边。对于大多数情况下(谓词通常是知道的恒定变量，然而subject/object是自由变量)，Wukong利用谓词索引开始进行图探索。对于那些查询是一个子图环的查询，三个Subjet/Object都是自由变量。Wukong根据基于cost的方法和一些启发式的选择一个探索顺序。对于一些罕见的情况，那些谓词都是不知道的情况下，Wukong从一个静态的(常量)的顶点进行图形探索（通过pred 已知的顶点相关联的谓词）。
2. Full-history Pruning
在探索查询的每一个阶段中，通过RDMA READ读取其他机器上的数据，进行裁剪。裁剪那些没有必要的冗余数据。

3. Migrating Execution or Data
对于一个查询阶段，如果有很少的顶点数据需要抓取从远程机器中，Wukong 使用一个本地执行的模式同步利用单边RDMA READ直接从远程顶点抓取数据。对于一个查询阶段，如果许多顶点需要被抓取。Wuong 利用一个Fork-Join 执行模式异步的分开查询计算到多个子查询在远程机器上。

![upload successful](\blog\images\pasted-45.png)

### Concurrent  Query Processing

Work-obliger work 窃取算法
邻近的Worker进程的查询超时时间（s.end < now）。如果是这样的话这个Worker可能在处理冗长的查询，因此后续的查询任务可能被延迟。在这种情况下，这个Worker从该Worker的工作对队列中窃取一个查询任务来处理。在逼迫相邻的woker(知道看到一个不忙的Worker)，Worker 进程持续通过从其中自己的工作队列中，持续处理自己的查询。持续处理自己的查询。