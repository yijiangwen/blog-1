# Elasticsearch面试专题
Elasticsearch面试专题。

## Elasticsearch的核心概念
near realTime（NRT）：近实时，从写入数据到数据可以搜索到有一个小延迟（一秒），基于es的搜索和分析可以达到秒级

集群：包含多个节点，每个节点是属于哪个集群是通过一个配置（集群名称，默认是elasticsearch）来决定的。

node节点：集群中的一个节点，节点的名称是默认随机分配的，在执行运维管理操作的时候节点名称很重要，默认节点会加入一个叫elasticsearch的集群。

index索引：包含一堆有相似结构的document文档数据，一个index代表了一类类似的或相同的document。

type类型：每个index里面可以有多个或一个type，6.x版本默认为_doc。

document和field：document是es中最小的数据单元，一个document可以是一条数据，通常用json数据结构表示，每个index下的type中可以存储多个document，一个document里面有多个field，每个field就是一个数据字段。

shard：单台机器无法存储大量的数据，es可以将一个index中的数据切分为多个shard，分布在多个机器上进行存储，有了shard可以进行横向扩展，进而可以存储更多的数据，让搜索和分析等操作分布到多台机器上去执行，提高吞吐量和性能，每个shard都是一个Lucene index。

replica：任何一个服务都可能出现故障或者宕机，此时shard的数据就有可能丢失，因此可以为每个shard建立replica副本，replica可以在shard发生故障时提供备用服务保证数据不丢失，多个replica还可以提供搜索和分析等操作的性能。建立index时一次性设置primary shard，不能进行修改，默认是5个，replica可以随便修改，默认是1个（这里的1个的意思是每个primary shard拥有一个replica shard），默认每个索引有10个shard，5个primary shard，5个replica shard，最小的高可用配置，默认是2台服务器。


## 什么是倒排索引？ 
倒排索引是搜索引擎的核心。搜索引擎的主要目标是在查找发生搜索条件的文档时提供快速搜索。
通俗解释一下就可以。
传统的我们的检索是通过文章，逐个遍历找到对应关键词的位置。
而倒排索引，是通过分词策略，形成了词和文章的映射关系表，这种词典+映射表即为倒排索引。

有了倒排索引，就能实现o（1）时间复杂度的效率检索文章了，极大的提高了检索效率。

## ElasticSearch中的集群、节点、索引、文档、类型是什么？

## Elasticsearch 索引文档的过程
协调节点默认使用文档ID参与计算（也支持通过routing），以便为路由提供合适的分片。
shard = hash(document_id) % (num_of_primary_shards)当分片所在的节点接收到来自协调节点的请求后，会将请求写入到Memory Buffer，然后定时（默认是每隔1秒）写入到Filesystem Cache，这个从Momery Buffer到Filesystem 　　Cache的过程就叫做refresh；

当然在某些情况下，存在Momery Buffer和Filesystem Cache的数据可能会丢失，ES是通过translog的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到translog中，当Filesystem cache中的数据写入到磁盘中时，才会清除掉，这个过程叫做flush；
在flush过程中，内存中的缓冲将被清除，内容被写入一个新段，段的fsync将创建一个新的提交点，并将内容刷新到磁盘，旧的translog将被删除并开始一个新的translog。
flush触发的时机是定时触发（默认30分钟）或者translog变得太大（默认为512M）时；

## Elasticsearch 读取文档的过程
可以通过 doc id 来查询，会根据 doc id 进行 hash，判断出来当时把 doc id 分配到了哪个 shard 上面去，从那个 shard 去查询。

* 客户端发送请求到任意一个 node，称为协调节点（coordinate node）。
* coordinate node 对 doc id 进行哈希路由，将请求转发到对应的 node，此时会使用 round-robin随机轮询算法，在 primary shard 以及其所有 replica 中随机选择一个，让读请求负载均衡。
* 接收请求的 node 返回 document 给 coordinate node。
* coordinate node 返回 document 给客户端。

写请求是写入 primary shard，然后同步给所有的 replica shard；读请求可以从 primary shard或replica shard 读取，采用的是随机轮询算法。

## Elasticsearch更新和删除文档的过程
删除和更新也都是写操作，但是Elasticsearch中的文档是不可变的，因此不能被删除或者改动以展示其变更；

磁盘上的每个段都有一个相应的.del文件。当删除请求发送后，文档并没有真的被删除，而是在.del文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del文件中被标记为删除的文档将不会被写入新段。

在新的文档被创建时，Elasticsearch会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉。

## Elasticsearch搜索的过程
搜索拆解为“query then fetch” 两个阶段。

query阶段的目的：定位到位置，但不取。

步骤拆解如下：
1. 假设一个索引数据有5主+1副本 共10分片，一次请求会命中（主或者副本分片中）的一个。
2. 每个分片在本地进行查询，结果返回到本地有序的优先队列中。
3. 第2）步骤的结果发送到协调节点，协调节点产生一个全局的排序列表。

查询阶段会计算文档的相关性得分用于排序。

fetch阶段的目的：取数据。
路由节点获取所有文档，返回给客户端。

## 在并发情况下Elasticsearch如果保证读写一致？
可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖，由应用层来处理具体的冲突；
另外对于写操作，一致性级别支持quorum/one/all，默认为quorum，即只有当大多数分片可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，分片将会在一个不同的节点上重建。
对于读操作，可以设置replication为sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置replication为async时，也可以通过设置搜索请求参数_preference为primary来查询主分片，确保文档是最新版本。

## Elasticsearch 海量数据如何调优？
### 1.设计阶段调优：
1. 根据业务增量需求，采取基于日期模板创建索引，通过roll over API滚动索引；
2. 使用别名进行索引管理；
3. 每天凌晨定时对索引做force_merge操作，以释放空间；
4. 采取冷热分离机制，热数据存储到SSD，提高检索效率；冷数据定期进行shrink操作，以缩减存储；
5. 采取curator进行索引的生命周期管理；
6. 仅针对需要分词的字段，合理的设置分词器；
7. Mapping阶段充分结合各个字段的属性，是否需要检索、是否需要存储等。

### 2.写入调优
1. 写入前副本数设置为0；
2. 写入前关闭refresh_interval设置为-1，禁用刷新机制；
3. 写入过程中：采取bulk批量写入；
4. 写入后恢复副本数和刷新间隔；
5. 尽量使用自动生成的id。

### 3.查询调优
1. 禁用wildcard；
2. 禁用批量terms（成百上千的场景）；
3. 充分利用倒排索引机制，能keyword类型尽量keyword；
4. 数据量大时候，可以先基于时间敲定索引再检索；
5. 设置合理的路由机制。

### 4.部署

1. 关闭缓存swap;
2. 堆内存设置为：Min（节点内存/2, 32GB）;
3. 设置最大文件句柄数；
4. 线程池+队列大小根据业务需要做调整；
5. 磁盘存储raid方式——存储有条件使用RAID10，增加单节点性能以及避免单节点存储故障。


