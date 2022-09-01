---
title: Elasticsearch学习记录
date: 2022-04-09 22:08:12
categories: ELK
tags: 
   - Elasticsearch
   - Kibana
urlname: elasticsearch-study
---

前言：目标是独立安装Elasticsearch、会使用RestClient的API操作索引和文档、会使用RestClient的API查询文档、会使用RestClient的API聚合数据、简单掌握Spring Data Elasticsearch使用。

## Elasticsearch介绍和安装

### 简介

#### Elastic

Elastic官网：https://www.elastic.co/cn/

Elastic有一条完整的产品线及解决方案：Elasticsearch、Kibana、Logstash等，前面说的三个就是大家常说的ELK技术栈。

#### Elasticsearch

Elasticsearch官网：https://www.elastic.co/cn/products/elasticsearch

Elasticsearch具备以下特点：

- 分布式，无需人工搭建集群（solr就需要人为配置，使用Zookeeper作为注册中心）
- Restful风格，一切API都遵循Rest原则，容易上手
- 近实时搜索，数据更新在Elasticsearch中几乎是完全同步的。

<!--more-->

#### 版本

需要虚拟机JDK1.8及以上，本篇博客是以 Elasticsearch7.12.1 为例。配套的Kibana的版本跟随Elasticsearch的版本一致。

#### 总结

什么是 elasticsearch ？

- 一个开源的分布式搜索引擎，可以用来实现搜索、日志统计、分析、系统监控等功能

什么是 elastic stack（ELK）？

- 是以 elasticsearch 为核心的技术栈，包括 beats、Logstash、kibana、elasticsearch

什么是 Lucene ？

- 是 Apache 的开源搜索引擎类库，提供了搜索引擎的核心 API

### 安装和配置

https://chanservy.github.io/posts/20220611/elasticsearch-install.html

#### 总结

分词器的作用是什么？

- 创建倒排索引时对文档分词
- 用户搜索时，对输入的内容分词

IK分词器有几种模式？

- ik_smart：智能切分，粗粒度
- ik_max_word：最细切分，细粒度

IK分词器如何拓展词条？如何停用词条？

- 利用config目录的IkAnalyzer.cfg.xml文件添加拓展词典和停用词典
- 在词典中添加拓展词条或者停用词条

### API

Elasticsearch提供了Rest风格的API，即http请求接口，而且也提供了各种语言的客户端API。

#### Rest风格API

文档地址：https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

#### 客户端API

Elasticsearch支持的客户端非常多：https://www.elastic.co/guide/en/elasticsearch/client/index.html

## 初识索引

### 正向索引

倒排索引的概念是基于 MySQL 这样的正向索引而言的。那么什么是正向索引呢？例如给下表（tb_goods）中的 id 创建索引：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720195531539.png)

对于数据库，它一般情况下都会基于 id 去创建一个索引，然后形成一个 B+ 树，那么根据 id 进行查询的速度则会非常的快，这种方式的索引就叫做正向索引。

如果是根据 id 查询，那么直接走索引，查询速度非常快。但是如果现在不是根据 id 字段进行搜索，而是一个普通的字段，可能这个字段比较长，比如标题字段，一般不会给它加索引，即便给它也加了索引，那如果进行的不是精确查询而是模糊查询，进行模糊匹配，即便这个字段有索引，那么索引也不会生效。

如果是基于 title 做模糊查询，只能是逐行扫描数据，流程如下：

1. 用户搜索数据，条件是title符合`"%手机%"`
2. 逐行获取数据，比如id为1的数据
3. 判断数据中的title是否符合用户搜索条件
4. 如果符合则放入结果集，不符合则丢弃。回到步骤1

逐行扫描，也就是全表扫描，随着数据量增加，其查询效率也会越来越低。当数据量达到数百万时，就是一场灾难。

### 倒排索引

倒排索引中有两个非常重要的概念：

- 文档（`Document`）：用来搜索的数据，其中的每一条数据就是一个文档。例如一个网页、一个商品信息
- 词条（`Term`）：对文档数据或用户搜索数据，利用某种算法分词，得到的具备含义的词语就是词条。例如：我是东北人，就可以分为：我、是、东北人、东北这样的几个词条。

**创建倒排索引**是对正向索引的一种特殊处理，流程如下：

- 将每一个文档的数据利用算法分词，得到一个个词条
- 创建表，每行数据包括词条、词条所在文档id、位置等信息
- 因为词条唯一性，可以给词条创建索引，例如hash表结构索引

将每一个文档的数据利用算法分词，得到一个个词条；创建表，每行数据包括词条、词条所在文档id、位置等信息；如果有重复的词条出现，在后面记录文档的 id 即可，这样可以确保倒排索引当中，词条字段绝对不会出现重复。将来根据词条查找的速度就会变得非常之快了。

如图：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720200457207.png)

倒排索引的**搜索流程**如下（以搜索"华为手机"为例）：

1. 用户输入条件`"华为手机"`进行搜索。
2. 对用户输入内容**分词**，得到词条：`华为`、`手机`。
3. 拿着词条在倒排索引中查找，可以得到包含词条的文档id：1、2、3。
4. 拿着文档id到正向索引中查找具体文档。

如图：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720201115192.png)

虽然要先查询倒排索引，再查询正向索引，但是无论是词条、还是文档 id 都建立了索引，查询速度非常快！无需全表扫描。倒排索引更擅长于基于文档的部分内容进行搜索，最经典的就是按照异常信息进行搜索，这也就是 ElasticSearch 便于分析日志的原因，Elasticsearch 底层就是基于倒排索引来查询的。

要注意倒排索引的两个重要细节：

- 倒排索引中的所有词条对应一个或多个文档（id）；
- 倒排索引中的词条**根据字典顺序升序排列**

### 正向和倒排

那么为什么一个叫做正向索引，一个叫做倒排索引呢？

- **正向索引**是最传统的，根据 id 索引的方式。但根据词条查询时，必须先逐条获取每个文档，然后判断文档中是否包含所需要的词条，是**根据文档找词条的过程**。

- 而**倒排索引**则相反，是先找到用户要搜索的词条，根据词条得到包含词条的文档的 id，然后根据 id 获取文档。是**根据词条找文档的过程**。

是不是恰好反过来了？那么两者方式的优缺点是什么呢？

**正向索引**：

- 优点：
  - 可以给多个字段创建索引
  - 根据索引字段搜索、排序速度非常快
- 缺点：
  - 根据非索引字段，或者索引字段中的部分词条查找时，只能全表扫描。

**倒排索引**：

- 优点：
  - 根据词条搜索、模糊搜索时，速度非常快
- 缺点：
  - 只能给词条创建索引，而不是字段
  - 无法根据字段做排序

### 基本概念

Elasticsearch 也是基于 Lucene 的全文检索库，本质也是存储数据，很多概念与 MySQL 类似的。

#### 文档和字段

elasticsearch 是面向**文档（Document）**存储的，可以是数据库中的一条商品数据，一个订单信息。文档数据会被序列化为 json 格式后存储在 elasticsearch 中：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720202707797.png)

而 Json 文档中往往包含很多的**字段（Field）**，类似于数据库中的列。

#### 索引和映射

**索引（Index）**，就是相同类型的文档的集合。

例如：

- 所有用户文档，就可以组织在一起，称为用户的索引；
- 所有商品的文档，可以组织在一起，称为商品的索引；
- 所有订单的文档，可以组织在一起，称为订单的索引；

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720203022172.png)

因此，我们可以把索引当做是数据库中的表。数据库的表会有约束信息，用来定义表的结构、字段的名称、类型等信息。因此，索引中就有**映射（mapping）**，是索引中的文档的字段约束信息，类似表的结构约束。

#### mysql与elasticsearch

我们统一的把mysql与elasticsearch的概念做一下对比：（7.x版本之后）

| **MySQL** | **Elasticsearch** | **说明**                                                     |
| --------- | ----------------- | ------------------------------------------------------------ |
| Table     | Index             | 索引(index)，就是文档的集合，类似数据库的表(table)           |
| Row       | Document          | 文档（Document），就是一条条的数据，类似数据库中的行（Row），文档都是JSON格式 |
| Column    | Field             | 字段（Field），就是JSON文档中的字段，类似数据库中的列（Column） |
| Schema    | Mapping           | Mapping（映射）是索引中文档的约束，例如字段类型约束。类似数据库的表结构（Schema） |
| SQL       | DSL               | DSL是elasticsearch提供的JSON风格的请求语句，用来操作elasticsearch，实现CRUD |

注：在Elasticsearch7.x版本之前，有type的概念，也即是类型，类型是elasticsearch模拟mysql中的table概念，以前的版本中一个索引下可以有不同的type类型，比如商品，订单，其数据格式不同。不过这会导致索引混乱！在elasticsearch 7.0.0版本必须使用单index，单type，多type结构则会完全移除。事实上6.2.4就已经是单index，单type了。最重要的是，未来版本中会完全移除type这个概念，直接就是`索引index即相当于MySQL中的table`。注：至今，本人elasticsearch的版本已更新至7.12.1。

既然和MySQL很多相似，是不是说，我们学习了elasticsearch就不再需要mysql了呢？并不是如此，两者各自有自己的擅长之处：

- Mysql：擅长事务类型操作，可以确保数据的安全和一致性
- Elasticsearch：擅长海量数据的搜索、分析、计算

因此在企业中，往往是两者结合使用：

- 对安全性要求较高的写操作，使用mysql实现
- 对查询性能要求较高的搜索需求，使用elasticsearch实现
- 两者再基于某种方式，实现数据的同步，保证一致性

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720203534945.png)

另外，在SolrCloud中，有一些集群相关的概念，在Elasticsearch也有类似的：

- 索引集（Indices，index的复数）：逻辑上的完整索引。
- 分片（shard）：数据拆分后的各个部分。
  - 单台机器无法存储大量数据，es 可以将一个索引中的数据切分为多个 shard，分布在多台服务器上存储。有了 shard 就可以横向扩展，存储更多数据，让搜索和分析等操作分布到多台服务器上去执行，提升吞吐量和性能。每个 shard 都是一个 lucene index。
- 副本（replica）：每个分片的复制。
  - 任何一个服务器随时可能故障或宕机，此时 shard 可能就会丢失，因此可以为每个 shard 创建多个 replica 副本。replica 可以在 shard 故障时提供备用服务，保证数据不丢失，多个 replica 还可以提升搜索操作的吞吐量和性能。primary shard（建立索引时一次设置，不能修改，默认 5 个），replica shard（随时修改数量，默认 1 个），默认每个索引 10 个 shard，5 个 primary shard，5个 replica shard，最小的高可用配置，是 2 台服务器。

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/8.png)

要注意的是：Elasticsearch本身就是分布式的，因此即便你只有一个节点，Elasticsearch默认也会对你的数据进行分片和副本操作，当你向集群添加新数据时，数据也会在新加入的节点中进行平衡。

这么说吧，shard 分为 primary shard 和 replica shard。而 primary shard 一般简称为 shard，而 replica shard 一般简称为 replica。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/50.png)

### es分布式架构原理

你搞一个索引，这个索引可以拆分成多个 `shard`，每个 shard 存储部分数据。拆分多个 shard 是有好处的，一是**支持横向扩展**，比如你数据量是 3T，3 个 shard，每个 shard 就 1T 的数据，若现在数据量增加到 4T，怎么扩展，很简单，重新建一个有 4 个 shard 的索引，将数据导进去；二是**提高性能**，数据分布在多个 shard，即多台服务器上，所有的操作，都会在多台机器上并行分布式执行，提高了吞吐量和性能。

接着就是这个 shard 的数据实际是有多个备份，就是说每个 shard 都有一个 `primary shard`，负责写入数据，但是还有几个 `replica shard`。`primary shard` 写入数据之后，会将数据同步到其他几个 `replica shard` 上去。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/52.png)

通过这个 replica 的方案，每个 shard 的数据都有多个备份，如果某个机器宕机了，没关系啊，还有别的数据副本在别的机器上呢。高可用了吧。

es 集群多个节点，会自动选举一个节点为 master 节点，这个 master 节点其实就是干一些管理的工作的，比如维护索引元数据、负责切换 primary shard 和 replica shard 身份等。要是 master 节点宕机了，那么会重新选举一个节点为 master 节点。

如果是非 master节点宕机了，那么会由 master 节点，让那个宕机节点上的 primary shard 的身份转移到其他机器上的 replica shard。接着你要是修复了那个宕机机器，重启了之后，master 节点会控制将缺失的 replica shard 分配过去，同步后续修改的数据之类的，让集群恢复正常。

说得更简单一点，就是说如果某个非 master 节点宕机了。那么此节点上的 primary shard 不就没了。那好，master 会让 primary shard 对应的 replica shard（在其他机器上）切换为 primary shard。如果宕机的机器修复了，修复后的节点也不再是 primary shard，而是 replica shard。

其实上述就是 ElasticSearch 作为分布式搜索引擎最基本的一个架构设计。

其实 es 的分布式架构原理和之前说过的 Kafka 的高可用分布式架构原理是极其相似的，搞一个 topic ，将 topic 分为多个 partition ，比如说数据过来了，partition1 存放一部分，partition2存放一部分。。。每个 partition 都只存放部分数据，这就凸显了分布式的理念。接着，每个 partition 都有多个 replica 副本，partition 自己也是个副本，然后这些个副本中，会选举出来一个 leader，其他的都是 follower，其实到这里，都是和 Kafka 极其相似的，每台机器上的 es 进程就相当于 Kafka 中的 broker ，索引就相当于 Kafka 的 topic ， shard 就相当于 Kafka 中的 partition ，primary shard 就相当于 partition 的 leader ， replica shard 就相当于 partition 的 follower。但是从这里开始，就有一个不同之处了，在 Kafka 中，读写消息都是针对于 partition 的 leader 的；在 es 中，写数据的时候是只向 primary shard 中写，读数据的时候，是 primary shard 和 replica shard 都可以读的。另外，在 es 中有一个选举 master node 主节点的因素。

### es读写数据的工作原理

#### es 写数据过程

- 客户端选择一个 node 发送请求过去，这个 node 就是 `coordinating node`（协调节点）。
- `coordinating node` 对 document 进行**路由**，将请求转发给对应的 node（有 primary shard）。
- 实际的 node 上的 `primary shard` 处理请求，然后将数据同步到 `replica shard`。
- `coordinating node` 如果发现 `primary shard` 和所有 `replica shard` 都搞定之后，就返回响应结果给客户端。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es-write-query/1.png)

> 图解：客户端写数据的时候，会任意挑一个进程去写，这个进程此时就会被称为协调节点，这个协调节点会对这条数据进行哈希，哈希之后就会知道这条数据属于哪个 shard 。如图，假设这条数据属于 shard01 ，但客户端比如说任意挑的是机器 2 上的 es 进程 02 节点，它就作为协调节点，写数据只能在 primary shard 上面写，一看 es 进程 02 上面的是 replica shard01 ，所以这个协调节点就会将请求转发给对应的有 primary shard01 的 node ，也就是机器 1 上的 es 进程 01 节点。然后这个 node 上的 primary shard01 处理请求，写入数据，然后将数据同步到其他的 replica shard01 上，协调节点如果发现 `primary shard` 和所有 `replica shard` 都搞定之后，就返回响应结果给客户端。

#### es 读数据过程

可以通过 `doc id` 来查询，会根据 `doc id` 进行 hash，判断出来当时把 `doc id` 分配到了哪个 shard 上面去，从那个 shard 去查询。

- 客户端发送请求到**任意**一个 node，成为 `coordinate node`。
- `coordinate node` 对 `doc id` 进行哈希路由，将请求转发到对应的 node，此时会使用 `round-robin` **随机轮询算法**，在 `primary shard` 以及其所有 replica 中随机选择一个，让读请求负载均衡。
- 接收请求的 node 返回 document 给 `coordinate node`。
- `coordinate node` 返回 document 给客户端。

#### es 搜索数据过程

es 最强大的是做全文检索，就是比如你有三条数据：

```
java真好玩儿啊
java好难学啊
j2ee特别牛
```

你根据 `java` 关键词来搜索，将包含 `java`的 `document` 给搜索出来。es 就会给你返回：java真好玩儿啊，java好难学啊。

- 客户端发送请求到一个 `coordinate node`。
- 协调节点将搜索请求转发到**所有**的 shard 对应的 `primary shard` 或 `replica shard`，都可以。
- query phase：每个 shard 将自己的搜索结果（其实就是一些 `doc id`）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果。
- fetch phase：接着由协调节点根据 `doc id` 去各个节点上**拉取实际**的 `document` 数据，最终返回给客户端。

> 写请求是写入 primary shard，然后同步给所有的 replica shard；读请求可以从 primary shard 或 replica shard 读取，采用的是随机轮询算法。

#### 写数据底层原理

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es-write-query/2.png)

先写入内存 buffer，在 buffer 里的时候数据是搜索不到的；同时将数据写入 translog 日志文件。

如果 buffer 快满了，或者到一定时间，就会将内存 buffer 数据 `refresh` 到一个新的 `segment file` 中，但是此时数据不是直接进入 `segment file` 磁盘文件，而是先进入 `os cache` 。这个过程就是 `refresh`。

每隔 1 秒钟，es 将 buffer 中的数据写入一个**新的** `segment file`，每秒钟会产生一个**新的磁盘文件** `segment file`，这个 `segment file` 中就存储最近 1 秒内 buffer 中写入的数据。

但是如果 buffer 里面此时没有数据，那当然不会执行 refresh 操作，如果 buffer 里面有数据，默认 1 秒钟执行一次 refresh 操作，刷入一个新的 segment file 中。

操作系统里面，磁盘文件其实都有一个东西，叫做 `os cache`，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 `os cache`，先进入操作系统级别的一个内存缓存中去。只要 `buffer` 中的数据被 refresh 操作刷入 `os cache`中，这个数据就可以被搜索到了。

为什么叫 es 是**准实时**的？ `NRT`，全称 `near real-time`。默认是每隔 1 秒 refresh 一次的，所以 es 是准实时的，因为写入的数据 1 秒之后才能被看到。可以通过 es 的 `restful api` 或者 `java api`，**手动**执行一次 refresh 操作，就是手动将 buffer 中的数据刷入 `os cache`中，让数据立马就可以被搜索到。只要数据被输入 `os cache` 中，buffer 就会被清空了，因为不需要保留 buffer 了，数据在 translog 里面已经持久化到磁盘去一份了。

重复上面的步骤，新的数据不断进入 buffer 和 translog，不断将 `buffer` 数据写入一个又一个新的 `segment file` 中去，每次 `refresh` 完 buffer 清空，translog 保留。随着这个过程推进，translog 会变得越来越大。当 translog 达到一定长度的时候，就会触发 `commit` 操作。

commit 操作发生第一步，就是将 buffer 中现有数据 `refresh` 到 `os cache` 中去，清空 buffer。然后，将一个 `commit point` 写入磁盘文件，里面标识着这个 `commit point` 对应的所有 `segment file`，同时强行将 `os cache` 中目前所有的数据都 `fsync` 到磁盘文件中去。最后**清空** 现有 translog 日志文件，重启一个 translog，此时 commit 操作完成。

这个 commit 操作叫做 `flush`。默认 30 分钟自动执行一次 `flush`，但如果 translog 过大，也会触发 `flush`。flush 操作就对应着 commit 的全过程，我们可以通过 es api，手动执行 flush 操作，手动将 os cache 中的数据 fsync 强刷到磁盘上去。

translog 日志文件的作用是什么？你执行 commit 操作之前，数据要么是停留在 buffer 中，要么是停留在 os cache 中，`无论是 buffer 还是 os cache 都是内存`，一旦这台机器死了，内存中的数据就全丢了。所以需要将数据对应的操作写入一个专门的日志文件 `translog` 中，一旦此时机器宕机，再次重启的时候，es 会自动读取 translog 日志文件中的数据，恢复到内存 buffer 和 os cache 中去。

translog 其实也是先写入 os cache 的，默认每隔 5 秒刷一次到磁盘中去，所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会**丢失** 5 秒钟的数据。但是这样性能比较好，最多丢 5 秒的数据。也可以将 translog 设置成每次写操作必须是直接 `fsync` 到磁盘，但是性能会差很多。

实际上你在这里，如果面试官没有问你 es 丢数据的问题，你可以在这里给面试官炫一把，你说，其实 es 第一是准实时的，数据写入 1 秒后可以搜索到；可能会丢失数据的。有 5 秒的数据，停留在 buffer、translog os cache、segment file os cache 中，而不在磁盘上，此时如果宕机，会导致 5 秒的**数据丢失**。

**总结一下**，数据先写入内存 buffer，然后每隔 1s，将数据 refresh 到 os cache，到了 os cache 数据就能被搜索到（所以我们才说 es 从写入到能被搜索到，中间有 1s 的延迟）。translog 其实也是先写入 os cache 的，每隔 5s，再将数据写入磁盘上的 translog 文件（这样如果机器宕机，内存数据全没，最多会有 5s 的数据丢失），translog 大到一定程度，或者默认每隔 30mins，会触发 commit 操作，第一步，就是将 buffer 中现有数据 `refresh` 到 `os cache` 中去，清空 buffer。然后，将一个 `commit point` 写入磁盘文件，里面标识着这个 `commit point` 对应的所有 `segment file`，同时强行将 `os cache` 中目前所有的数据都 `fsync` 到 `segment file` 磁盘文件中去。最后**清空** 现有 translog 日志文件，重启一个 translog，至此 commit 操作完成，成功将缓冲区的数据都 flush 到 segment file 磁盘文件中。

> 数据写入 segment file 之后，同时就建立好了倒排索引。

#### 删除/更新数据底层原理

如果是删除操作，commit 的时候会生成一个 `.del` 文件，里面将某个 doc 标识为 `deleted` 状态，那么搜索的时候根据 `.del` 文件就知道这个 doc 是否被删除了。

如果是更新操作，就是将原来的 doc 标识为 `deleted` 状态，然后新写入一条数据。

buffer 每 refresh 一次，就会产生一个 `segment file`，所以默认情况下是 1 秒钟一个 `segment file`，这样下来 `segment file` 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 `segment file` 合并成一个，同时这里会将标识为 `deleted` 的 doc 给**物理删除掉**，然后将新的 `segment file` 写入磁盘，这里会写一个 `commit point`，标识所有新的 `segment file`，然后打开 `segment file` 供搜索使用，同时删除旧的 `segment file`。

### 性能优化的杀手锏——filesystem cache

你往 es 里写的数据，实际上都写到磁盘文件里去了，**查询的时候**，操作系统会将磁盘文件里的数据自动缓存到 `filesystem cache` 里面去。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es-write-query/3.png)

es 的搜索引擎严重依赖于底层的 `filesystem cache`，你如果给 `filesystem cache` 更多的内存，尽量让内存可以容纳所有的 `idx segment file `索引数据文件，那么你搜索的时候就基本都是走内存的，性能会非常高。

## 索引操作

索引就类似数据库中的表，mapping映射就类似表的结构。我们要向es中存储数据，必须先创建“表结构”。

### mapping映射属性

mapping是对索引中的文档的约束，常见的mapping属性包括：

- type：字段数据类型，常见的简单类型有：
  - 字符串：text（可分词的文本）、keyword（精确值，例如：品牌、国家、ip地址）
  - 数值：long、integer、short、byte、double、float、
  - 布尔：boolean
  - 日期：date
  - 对象：object
  - 嵌入式：nested
- index：是否创建索引，默认为true
- analyzer：使用哪种分词器
- properties：该字段的子字段
- store：是否存储，默认为false

例如下面的json文档：

```json
{
    "age": 21,
    "weight": 52.1,
    "isMarried": false,
    "info": "黑马程序员Java讲师",
    "email": "zy@itcast.cn",
    "score": [99.1, 99.5, 98.9],
    "name": {
        "firstName": "云",
        "lastName": "赵"
    }
}
```

对应的每个字段映射（mapping）：

- age：类型为 integer；参与搜索，因此需要index为true；无需分词器
- weight：类型为float；参与搜索，因此需要index为true；无需分词器
- isMarried：类型为boolean；参与搜索，因此需要index为true；无需分词器
- info：类型为字符串，需要分词，因此是text；参与搜索，因此需要index为true；分词器可以用ik_smart
- email：类型为字符串，但是不需要分词，因此是keyword；不参与搜索，因此需要index为false；无需分词器
- score：虽然是数组，但是我们只看元素的类型，类型为float；参与搜索，因此需要index为true；无需分词器
- name：类型为object，需要定义多个子属性
  - name.firstName；类型为字符串，但是不需要分词，因此是keyword；参与搜索，因此需要index为true；无需分词器
  - name.lastName；类型为字符串，但是不需要分词，因此是keyword；参与搜索，因此需要index为true；无需分词器

创建索引，最关键的是mapping映射，而mapping映射要考虑的信息包括：

- 字段名
- 字段数据类型
- 是否参与搜索
- 是否需要分词
- 如果分词，分词器是什么？

其中：

- 字段名、字段数据类型，可以参考数据表结构的名称和类型
- 是否参与搜索要分析业务来判断，例如图片地址，就无需参与搜索
- 是否分词呢要看内容，内容如果是一个整体就无需分词，反之则要分词
- 分词器，我们可以统一使用ik_max_word

举例来看下酒店数据的索引库结构：

```json
PUT /hotel
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name":{
        "type": "text",
        "analyzer": "ik_max_word",
        "copy_to": "all"
      },
      "address":{
        "type": "keyword",
        "index": false
      },
      "price":{
        "type": "integer"
      },
      "score":{
        "type": "integer"
      },
      "brand":{
        "type": "keyword",
        "copy_to": "all"
      },
      "city":{
        "type": "keyword",
        "copy_to": "all"
      },
      "starName":{
        "type": "keyword"
      },
      "business":{
        "type": "keyword"
      },
      "location":{
        "type": "geo_point"
      },
      "pic":{
        "type": "keyword",
        "index": false
      },
      "all":{
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}
```

几个特殊字段说明：

- location：地理坐标，里面包含精度、纬度
- all：一个组合字段，其目的是将多字段的值利用copy_to合并，提供给用户搜索

地理坐标说明：

![image-20210720222110126](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720222110126.png)

copy_to说明：

![image-20210720222221516](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720222221516.png)

#### 案例分析

> 项目经验，案例分析。

**检索业务-es中sku的存储结构分析**

（只有上架的商品存储到Elasticsearch中才能被检索）

> 分析-存储商品的什么信息到 es 中

1. 需要保存 sku 信息
   - 当搜索商品名时，查询的是 sku 的标题 sku_title；
   - 可能通过 sku 的标题、销量、价格区间检索

2. 需要保存品牌、分类等信息
   - 点击分类，检索分类下的所有信息
   - 点击品牌，检索品牌下的商品信息

3. 需要保存 spu 信息
   - 选择规格，检索共有这些规格的商品

> 分析-怎么设计存储结构来保存数据

1. 方案1-空间换时间

   ```json
   {
       skuId:1
       spuId:11
       skyTitile:华为xx
       price:999
       saleCount:99
       attrs:[
           {尺寸:5存},
           {CPU:高通945},
           {分辨率:全高清}
   	]
   }
   # 缺点：会产生冗余字段，对于相同类型的商品，attrs 属性字段会重复，空间占用大
   # 好处：方便检索
   ```

2. 方案2-时间换空间

   ```json
   sku索引
   {
       skuId:1
       spuId:11
   }
   attr索引
   {
       spuId:11
       attrs:[
           {尺寸:5寸},
           {CPU:高通945},
           {分辨率:全高清}
   	]
   }
   # 缺点：选择公共属性attr时,会检索当前属性的所有商品分类，然后再查询当前商品分类的所有可能属性；导致耗时变长。
   # 好处：空间利用率高
   ```

举例，mapping结构字段说明：

```json
"mappings": {
  "properties": {
    "skuId": { "type": "long" },
    "spuId": { "type": "keyword" }, # 精确检索，不分词
    "skuTitle": {
      "type": "text", # 全文检索
      "analyzer": "ik_smart" # 分词器
    },
    "skuPrice": { "type": "keyword" },
    "skuImg": {
      "type": "keyword",
      "index": false, # false 不可被检索
      "doc_values": false # false 不可被聚合
    },
    "saleCount":{ "type":"long" }, # 商品销量
    "hasStock": { "type": "boolean" }, # 商品是否有库存
    "hotScore": { "type": "long"  }, # 商品热度评分
    "brandId":  { "type": "long" }, # 品牌id
    "catalogId": { "type": "long"  }, # 分类id
    "brandName": {	# 品牌名，只用来查看，不用来检索和聚合
      "type": "keyword",
      "index": false,
      "doc_values": false
    },
    "brandImg":{	# 品牌图片，只用来查看，不用来检索和聚合
      "type": "keyword",
      "index": false,
      "doc_values": false
    },
    "catalogName": {	# 分类名，只用来查看，不用来检索和聚合
      "type": "keyword",
      "index": false,
      "doc_values": false
    },
    "attrs": {	# 属性对象
      "type": "nested",	# 嵌入式，内部属性
      "properties": {
        "attrId": {"type": "long"  },
        "attrName": {	# 属性名
          "type": "keyword",
          "index": false,
          "doc_values": false
        },
        "attrValue": { "type": "keyword" }	# 属性值
      }
    }
  }
}
```

#### 关于 nested 类型

官网：[Nested field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)，nested 类型用于 JSON 对象数组。

- Object 数据类型的数组会被扁平化处理为一个简单的键与值的列表，即对象的相同属性会放到同一个数组中，在检索时会出现错误。参考官网：[How arrays of objects are flattened](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html#nested-arrays-flattening-objects)
- 对于 Object 类型的数组，要使用 nested 字段类型。参考官网：[Using nested fields for arrays of objects](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html#nested-fields-array-objects)

ElasticSearch-数组的扁平化处理：

举例：

```json
PUT my_index/_doc/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

在索引index中，存入user的数据，最终 es 会将上述数据，扁平化处理，实际存储如下这样子：

```json
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```

很明显，数据存储成这样子，丢失了first 和 last 之间关系。从这样的存储中，我们无法确定，first 为 “alice” 的对应的 last 是 “smith” 还是 “white”。因此，如果数组里存储的是对象，那么数组的类型应该是 nested，这样，在将对象数据存入数组中时，对象便不会被扁平化处理。

例如执行如下查询：

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
```

查询到的结果：

```json
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.5753642,
    "hits" : [
      {
        "_index" : "my_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.5753642,
        "_source" : {
          "group" : "fans",
          "user" : [
            {
              "first" : "John",
              "last" : "Smith"
            },
            {
              "first" : "Alice",
              "last" : "White"
            }
          ]
        }
      }
    ]
  }
}
```

虽然查询到了数据，但这样的数据并不是我们想要的。因为通过 `"first" 为 "Alice"` 并且 `"last" 为 "Smith"` 这样的条件，不应该查询到数据，（约翰史密斯、爱丽丝怀特）。之所以查询到数据，是因为数组中存储的对象被扁平化处理了。

因此需要修改索引的mapping信息，将user数组的类型定义为 nested 并存入数据，重新执行检索。

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested" 
      }
    }
  }
}

PUT my_index/_doc/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

执行同样查询：

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
```

这次获取到的结果为空：

```json
{
  "took" : 11,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

这样的结果才是我们想要的，这样防止了 frist 和 last 之间关系的丢失。

### 索引的CRUD

#### 创建索引和映射

基本语法：

- 请求方式：PUT
- 请求路径：/索引名，可以自定义
- 请求参数：mapping映射

格式：

```json
PUT /索引名称
{
  "mappings": {
    "properties": {
      "字段名":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "字段名2":{
        "type": "keyword",
        "index": "false"
      },
      "字段名3":{
        "properties": {
          "子字段": {
            "type": "keyword"
          }
        }
      },
      // ...略
    }
  }
}
```

示例：

```json
PUT /heima
{
  "mappings": {
    "properties": {
      "info":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "email":{
        "type": "keyword",
        "index": "falsae"
      },
      "name":{
        "properties": {
          "firstName": {
            "type": "keyword"
          }
        }
      },
      // ... 略
    }
  }
}
```

#### 查询索引

基本语法：

- 请求方式：GET

- 请求路径：/索引名

- 请求参数：无

格式：

```
GET /索引名
```

示例：

![image-20210720211019329](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720211019329.png)

#### 修改索引

倒排索引结构虽然不复杂，但是一旦数据结构改变（比如改变了分词器），就需要重新创建倒排索引，这简直是灾难。因此索引**一旦创建，无法修改mapping**。虽然无法修改mapping中已有的字段，但是却允许添加新的字段到mapping中，因为不会对倒排索引产生影响。

**语法说明**：

```json
PUT /索引名/_mapping
{
  "properties": {
    "新字段名":{
      "type": "integer"
    }
  }
}
```

**示例**：

![image-20210720212357390](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720212357390.png)

#### 删除索引

**语法：**

- 请求方式：DELETE

- 请求路径：/索引名

- 请求参数：无

**格式：**

```
DELETE /索引名
```

在kibana中测试：

![image-20210720212123420](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720212123420.png)

#### 总结

索引操作有哪些？

- 创建索引：PUT /索引名
- 查询索引：GET /索引名
- 删除索引：DELETE /索引名
- 添加字段：PUT /索引名/_mapping

#### 查看索引的配置

Get请求可以帮我们查看索引信息，格式：

```shell
GET /索引名
```

![22](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/22.png)

或者，我们可以使用*来查询所有索引的配置：

![23](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/23.png)

#### 查看映射关系

语法格式：

```fallback
GET /索引名/_mapping
```

示例：

```fallback
GET /csy/_mapping
```

响应：

```json
{
  "csy": {
    "mappings": {
      "goods": {
        "properties": {
          "images": {
            "type": "keyword",
            "index": false
          },
          "price": {
            "type": "float"
          },
          "title": {
            "type": "text",
            "analyzer": "ik_max_word"
          }
        }
      }
    }
  }
}
```

#### 字段属性详解

##### type

Elasticsearch中支持的数据类型非常丰富：

![24](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/24.png)

我们说几个关键的：

- String类型，又分两种：

  - text：可分词，不可参与聚合
  - keyword：不可分词，数据会作为完整字段进行匹配，可以参与聚合

- Numerical：数值类型，分两类

  - 基本数据类型：long、interger、short、byte、double、float、half_float
  - 浮点数的高精度类型：scaled_float
    - 需要指定一个精度因子，比如10或100。elasticsearch会把真实值乘以这个因子后存储，取出时再还原。

- Date：日期类型

  elasticsearch可以对日期格式化为字符串存储，但是建议我们存储为毫秒值，存储为long，节省空间。

- 如果存的是对象

  比如：{girl: {name: "rose",age: 22}}

  会处理成两个字段：`girl.name`和`girl.age`。

##### index

index影响字段的索引情况。

- true：字段会被索引，则可以用来进行搜索。默认值就是true
- false：字段不会被索引，不能用来搜索

index的默认值就是true，也就是说你不进行任何配置，所有字段都会被索引。

但是有些字段是我们不希望被索引的，比如商品的图片信息，就需要手动设置index为false。

##### store

是否将数据进行额外存储。

在lucene和solr中，我们知道如果一个字段的store设置为false，那么在文档列表中就不会有这个字段的值，用户的搜索结果中不会显示出来。

但是在Elasticsearch中，即便store设置为false，也可以搜索到结果。

原因是Elasticsearch在创建文档索引时，会将文档中的原始数据备份，保存到一个叫做`_source`的属性中。而且我们可以通过过滤`_source`来选择哪些要显示，哪些不显示。

而如果设置store为true，就会在`_source`以外额外存储一份数据，多余，因此一般我们都会将store设置为false，事实上，`store的默认值就是false`。

### 数据迁移

先创建出 new_twitter 的正确映射。然后使用如下方式进行数据迁移

```json
POST _reindex [固定写法]
{ 
    "source": { 
        "index": "twitter"
	},
    "dest": { 
        "index": "new_twitter"
	}
}
```



## RestClient操作索引

ES官方提供了各种不同语言的客户端，用来操作ES。这些客户端的本质就是组装DSL语句，通过http请求发送给ES。官方文档地址：https://www.elastic.co/guide/en/elasticsearch/client/index.html

其中的Java Rest Client又包括两种：

- Java Low Level Rest Client
- Java High Level Rest Client

![image-20210720214555863](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720214555863.png)

我们学习的是Java HighLevel Rest Client客户端API。

### 导入数据

数据结构如下：

```sql
CREATE TABLE `tb_hotel` (
  `id` bigint(20) NOT NULL COMMENT '酒店id',
  `name` varchar(255) NOT NULL COMMENT '酒店名称；例：7天酒店',
  `address` varchar(255) NOT NULL COMMENT '酒店地址；例：航头路',
  `price` int(10) NOT NULL COMMENT '酒店价格；例：329',
  `score` int(2) NOT NULL COMMENT '酒店评分；例：45，就是4.5分',
  `brand` varchar(32) NOT NULL COMMENT '酒店品牌；例：如家',
  `city` varchar(32) NOT NULL COMMENT '所在城市；例：上海',
  `star_name` varchar(16) DEFAULT NULL COMMENT '酒店星级，从低到高分别是：1星到5星，1钻到5钻',
  `business` varchar(255) DEFAULT NULL COMMENT '商圈；例：虹桥',
  `latitude` varchar(32) NOT NULL COMMENT '纬度；例：31.2497',
  `longitude` varchar(32) NOT NULL COMMENT '经度；例：120.3925',
  `pic` varchar(255) DEFAULT NULL COMMENT '酒店图片；例:/img/1.jpg',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 导入项目

项目结构如图：

![image-20210720220647541](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720220647541.png)

### mapping映射分析

创建索引库，最关键的是mapping映射，而mapping映射要考虑的信息包括：

- 字段名
- 字段数据类型
- 是否参与搜索
- 是否需要分词
- 如果分词，分词器是什么？

其中：

- 字段名、字段数据类型，可以参考数据表结构的名称和类型
- 是否参与搜索要分析业务来判断，例如图片地址，就无需参与搜索
- 是否分词呢要看内容，内容如果是一个整体就无需分词，反之则要分词
- 分词器，我们可以统一使用ik_max_word

来看下酒店数据的索引库结构:

```json
PUT /hotel
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name":{
        "type": "text",
        "analyzer": "ik_max_word",
        "copy_to": "all"
      },
      "address":{
        "type": "keyword",
        "index": false
      },
      "price":{
        "type": "integer"
      },
      "score":{
        "type": "integer"
      },
      "brand":{
        "type": "keyword",
        "copy_to": "all"
      },
      "city":{
        "type": "keyword",
        "copy_to": "all"
      },
      "starName":{
        "type": "keyword"
      },
      "business":{
        "type": "keyword"
      },
      "location":{
        "type": "geo_point"
      },
      "pic":{
        "type": "keyword",
        "index": false
      },
      "all":{
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}
```



几个特殊字段说明：

- location：地理坐标，里面包含精度、纬度
- all：一个组合字段，其目的是将多字段的值 利用copy_to合并，提供给用户搜索

地理坐标说明：

![image-20210720222110126](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720222110126.png)

copy_to说明：

![image-20210720222221516](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720222221516.png)



### 初始化RestClient

在elasticsearch提供的API中，与elasticsearch一切交互都封装在一个名为RestHighLevelClient的类中，必须先完成这个对象的初始化，建立与elasticsearch的连接。

分为三步：

1）引入es的RestHighLevelClient依赖：

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```

2）因为SpringBoot默认的ES版本是7.6.2，所以我们需要覆盖默认的ES版本：

```xml
<properties>
    <java.version>1.8</java.version>
    <elasticsearch.version>7.12.1</elasticsearch.version>
</properties>
```

3）初始化RestHighLevelClient：

初始化的代码如下：

```java
RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(
        HttpHost.create("http://192.168.150.101:9200")
));
```

这里为了单元测试方便，我们创建一个测试类HotelIndexTest，然后将初始化的代码编写在@BeforeEach方法中：

```java
package cn.itcast.hotel;

import org.apache.http.HttpHost;
import org.elasticsearch.client.RestHighLevelClient;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;

public class HotelIndexTest {
    private RestHighLevelClient client;

    @BeforeEach
    void setUp() {
        this.client = new RestHighLevelClient(RestClient.builder(
                HttpHost.create("http://192.168.150.101:9200")
        ));
    }

    @AfterEach
    void tearDown() throws IOException {
        this.client.close();
    }
}
```

### 创建索引库

#### 代码解读

创建索引库的API如下：

![image-20210720223049408](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720223049408.png)

代码分为三步：

- 1）创建Request对象。因为是创建索引库的操作，因此Request是CreateIndexRequest。
- 2）添加请求参数，其实就是DSL的JSON参数部分。因为json字符串很长，这里是定义了静态字符串常量MAPPING_TEMPLATE，让代码看起来更加优雅。
- 3）发送请求，client.indices()方法的返回值是IndicesClient类型，封装了所有与索引库操作有关的方法。

#### 完整示例

定义mapping映射的JSON字符串常量：

```java
package cn.itcast.hotel.constants;

public class HotelConstants {
    public static final String MAPPING_TEMPLATE = "{\n" +
            "  \"mappings\": {\n" +
            "    \"properties\": {\n" +
            "      \"id\": {\n" +
            "        \"type\": \"keyword\"\n" +
            "      },\n" +
            "      \"name\":{\n" +
            "        \"type\": \"text\",\n" +
            "        \"analyzer\": \"ik_max_word\",\n" +
            "        \"copy_to\": \"all\"\n" +
            "      },\n" +
            "      \"address\":{\n" +
            "        \"type\": \"keyword\",\n" +
            "        \"index\": false\n" +
            "      },\n" +
            "      \"price\":{\n" +
            "        \"type\": \"integer\"\n" +
            "      },\n" +
            "      \"score\":{\n" +
            "        \"type\": \"integer\"\n" +
            "      },\n" +
            "      \"brand\":{\n" +
            "        \"type\": \"keyword\",\n" +
            "        \"copy_to\": \"all\"\n" +
            "      },\n" +
            "      \"city\":{\n" +
            "        \"type\": \"keyword\",\n" +
            "        \"copy_to\": \"all\"\n" +
            "      },\n" +
            "      \"starName\":{\n" +
            "        \"type\": \"keyword\"\n" +
            "      },\n" +
            "      \"business\":{\n" +
            "        \"type\": \"keyword\"\n" +
            "      },\n" +
            "      \"location\":{\n" +
            "        \"type\": \"geo_point\"\n" +
            "      },\n" +
            "      \"pic\":{\n" +
            "        \"type\": \"keyword\",\n" +
            "        \"index\": false\n" +
            "      },\n" +
            "      \"all\":{\n" +
            "        \"type\": \"text\",\n" +
            "        \"analyzer\": \"ik_max_word\"\n" +
            "      }\n" +
            "    }\n" +
            "  }\n" +
            "}";
}
```



在hotel-demo中的HotelIndexTest测试类中，编写单元测试，实现创建索引：

```java
@Test
void createHotelIndex() throws IOException {
    // 1.创建Request对象
    CreateIndexRequest request = new CreateIndexRequest("hotel");
    // 2.准备请求的参数：DSL语句
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    // 3.发送请求
    client.indices().create(request, RequestOptions.DEFAULT);
}
```



### 删除索引库

删除索引库的DSL语句非常简单：

```json
DELETE /hotel
```

与创建索引库相比：

- 请求方式从PUT变为DELTE
- 请求路径不变
- 无请求参数

所以代码的差异，注意体现在Request对象上。依然是三步走：

- 1）创建Request对象。这次是DeleteIndexRequest对象
- 2）准备参数。这里是无参
- 3）发送请求。改用delete方法

在hotel-demo中的HotelIndexTest测试类中，编写单元测试，实现删除索引：

```java
@Test
void testDeleteHotelIndex() throws IOException {
    // 1.创建Request对象
    DeleteIndexRequest request = new DeleteIndexRequest("hotel");
    // 2.发送请求
    client.indices().delete(request, RequestOptions.DEFAULT);
}
```



### 判断索引库是否存在

判断索引库是否存在，本质就是查询，对应的DSL是：

```json
GET /hotel
```



因此与删除的Java代码流程是类似的。依然是三步走：

- 1）创建Request对象。这次是GetIndexRequest对象
- 2）准备参数。这里是无参
- 3）发送请求。改用exists方法

```java
@Test
void testExistsHotelIndex() throws IOException {
    // 1.创建Request对象
    GetIndexRequest request = new GetIndexRequest("hotel");
    // 2.发送请求
    boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
    // 3.输出
    System.err.println(exists ? "索引库已经存在！" : "索引库不存在！");
}
```

### 总结

JavaRestClient操作elasticsearch的流程基本类似。核心是client.indices()方法来获取索引库的操作对象。

索引库操作的基本步骤：

- 初始化RestHighLevelClient
- 创建XxxIndexRequest。XXX是Create、Get、Delete
- 准备DSL（ Create时需要，其它是无参）
- 发送请求。调用RestHighLevelClient#indices().xxx()方法，xxx是create、exists、delete



## 文档操作

### 新增文档

**语法：**

```json
POST /索引名/_doc/文档id
{
    "字段1": "值1",
    "字段2": "值2",
    "字段3": {
        "子属性1": "值3",
        "子属性2": "值4"
    },
    // ...
}
```

**示例：**

```json
POST /heima/_doc/1
{
    "info": "黑马程序员Java讲师",
    "email": "zy@itcast.cn",
    "name": {
        "firstName": "云",
        "lastName": "赵"
    }
}
```

**响应：**

![image-20210720212933362](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720212933362.png)

### 查询文档

根据rest风格，新增是post，查询应该是get，不过查询一般都需要条件，这里我们把文档id带上，这里是按照id查询，一些复杂查询后面说。

**语法：**

```json
GET /{索引名称}/_doc/{id}
```

**通过kibana查看数据：**

```json
GET /heima/_doc/1
```

**查看结果：**

![image-20210720213345003](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720213345003.png)

### 删除文档

删除使用DELETE请求，同样，需要根据id进行删除：

**语法：**

```json
DELETE /{索引名}/_doc/id值
```

**示例：**

```json
# 根据id删除数据
DELETE /heima/_doc/1
```

**结果：**

![image-20210720213634918](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210720213634918.png)

### 修改文档

修改有两种方式：

- 全量修改：直接覆盖原来的文档
- 增量修改：修改文档中的部分字段

#### 全量修改

全量修改是覆盖原来的文档，其本质是：

- 根据指定的id删除文档
- 新增一个相同id的文档

**注意**：如果根据id删除时，id不存在，第二步的新增也会执行，也就从修改变成了新增操作了。

**语法：**

```json
PUT /{索引名}/_doc/文档id
{
    "字段1": "值1",
    "字段2": "值2",
    // ... 略
}

```

**示例：**

```json
PUT /heima/_doc/1
{
    "info": "黑马程序员高级Java讲师",
    "email": "zy@itcast.cn",
    "name": {
        "firstName": "云",
        "lastName": "赵"
    }
}
```

#### 增量修改

增量修改是只修改指定id匹配的文档中的部分字段。

**语法：**

```json
POST /{索引名}/_update/文档id
{
    "doc": {
         "字段名": "新的值",
    }
}
```

**示例：**

```json
POST /heima/_update/1
{
  "doc": {
    "email": "ZhaoYun@itcast.cn"
  }
}
```

### 总结

文档操作有哪些？

- 创建文档：POST /{索引名}/_doc/文档id   { json文档 }
- 查询文档：GET /{索引名}/_doc/文档id
- 删除文档：DELETE /{索引名}/_doc/文档id
- 修改文档：
  - 全量修改：PUT /{索引名}/_doc/文档id   { json文档 }
  - 增量修改：POST /{索引名}/_update/文档id   { "doc": {字段}}



## RestClient操作文档

为了与索引库操作分离，我们再次参加一个测试类，做两件事情：

- 初始化RestHighLevelClient
- 我们的酒店数据在数据库，需要利用IHotelService去查询，所以注入这个接口

```java
package cn.itcast.hotel;

import cn.itcast.hotel.pojo.Hotel;
import cn.itcast.hotel.service.IHotelService;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.io.IOException;
import java.util.List;

@SpringBootTest
public class HotelDocumentTest {
    @Autowired
    private IHotelService hotelService;

    private RestHighLevelClient client;

    @BeforeEach
    void setUp() {
        this.client = new RestHighLevelClient(RestClient.builder(
                HttpHost.create("http://192.168.150.101:9200")
        ));
    }

    @AfterEach
    void tearDown() throws IOException {
        this.client.close();
    }
}

```

### 新增文档

我们要将数据库的酒店数据查询出来，写入elasticsearch中。

#### 索引库实体类

数据库查询后的结果是一个Hotel类型的对象。结构如下：

```java
@Data
@TableName("tb_hotel")
public class Hotel {
    @TableId(type = IdType.INPUT)
    private Long id;
    private String name;
    private String address;
    private Integer price;
    private Integer score;
    private String brand;
    private String city;
    private String starName;
    private String business;
    private String longitude;
    private String latitude;
    private String pic;
}
```

与我们的索引库结构存在差异：

- longitude和latitude需要合并为location

因此，我们需要定义一个新的类型，与索引库结构吻合：

```java
package cn.itcast.hotel.pojo;

import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
public class HotelDoc {
    private Long id;
    private String name;
    private String address;
    private Integer price;
    private Integer score;
    private String brand;
    private String city;
    private String starName;
    private String business;
    private String location;
    private String pic;

    public HotelDoc(Hotel hotel) {
        this.id = hotel.getId();
        this.name = hotel.getName();
        this.address = hotel.getAddress();
        this.price = hotel.getPrice();
        this.score = hotel.getScore();
        this.brand = hotel.getBrand();
        this.city = hotel.getCity();
        this.starName = hotel.getStarName();
        this.business = hotel.getBusiness();
        this.location = hotel.getLatitude() + ", " + hotel.getLongitude();
        this.pic = hotel.getPic();
    }
}

```

#### 语法说明

新增文档的DSL语句如下：

```json
POST /{索引库名}/_doc/1
{
    "name": "Jack",
    "age": 21
}
```

对应的java代码如图：

![image-20210720230027240](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720230027240.png)



可以看到与创建索引库类似，同样是三步走：

- 1）创建Request对象
- 2）准备请求参数，也就是DSL中的JSON文档
- 3）发送请求

变化的地方在于，这里直接使用client.xxx()的API，不再需要client.indices()了。



#### 完整代码

我们导入酒店数据，基本流程一致，但是需要考虑几点变化：

- 酒店数据来自于数据库，我们需要先查询出来，得到hotel对象
- hotel对象需要转为HotelDoc对象
- HotelDoc需要序列化为json格式

因此，代码整体步骤如下：

- 1）根据id查询酒店数据Hotel
- 2）将Hotel封装为HotelDoc
- 3）将HotelDoc序列化为JSON
- 4）创建IndexRequest，指定索引库名和id
- 5）准备请求参数，也就是JSON文档
- 6）发送请求

在hotel-demo的HotelDocumentTest测试类中，编写单元测试：

```java
@Test
void testAddDocument() throws IOException {
    // 1.根据id查询酒店数据
    Hotel hotel = hotelService.getById(61083L);
    // 2.转换为文档类型
    HotelDoc hotelDoc = new HotelDoc(hotel);
    // 3.将HotelDoc转json
    String json = JSON.toJSONString(hotelDoc);

    // 1.准备Request对象
    IndexRequest request = new IndexRequest("hotel").id(hotelDoc.getId().toString());
    // 2.准备Json文档
    request.source(json, XContentType.JSON);
    // 3.发送请求
    client.index(request, RequestOptions.DEFAULT);
}
```



### 查询文档

#### 语法说明

查询的DSL语句如下：

```json
GET /hotel/_doc/{id}
```

非常简单，因此代码大概分两步：

- 准备Request对象
- 发送请求

不过查询的目的是得到结果，解析为HotelDoc，因此难点是结果的解析。完整代码如下：

![image-20210720230811674](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720230811674.png)

可以看到，结果是一个JSON，其中文档放在一个`_source`属性中，因此解析就是拿到`_source`，反序列化为Java对象即可。

与之前类似，也是三步走：

- 1）准备Request对象。这次是查询，所以是GetRequest
- 2）发送请求，得到结果。因为是查询，这里调用client.get()方法
- 3）解析结果，就是对JSON做反序列化



#### 完整代码

在hotel-demo的HotelDocumentTest测试类中，编写单元测试：

```java
@Test
void testGetDocumentById() throws IOException {
    // 1.准备Request
    GetRequest request = new GetRequest("hotel", "61082");
    // 2.发送请求，得到响应
    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    // 3.解析响应结果
    String json = response.getSourceAsString();

    HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
    System.out.println(hotelDoc);
}
```





### 删除文档

删除的DSL为是这样的：

```json
DELETE /hotel/_doc/{id}
```

与查询相比，仅仅是请求方式从DELETE变成GET，可以想象Java代码应该依然是三步走：

- 1）准备Request对象，因为是删除，这次是DeleteRequest对象。要指定索引库名和id
- 2）准备参数，无参
- 3）发送请求。因为是删除，所以是client.delete()方法



在hotel-demo的HotelDocumentTest测试类中，编写单元测试：

```java
@Test
void testDeleteDocument() throws IOException {
    // 1.准备Request
    DeleteRequest request = new DeleteRequest("hotel", "61083");
    // 2.发送请求
    client.delete(request, RequestOptions.DEFAULT);
}
```







### 修改文档

#### 语法说明

修改我们讲过两种方式：

- 全量修改：本质是先根据id删除，再新增
- 增量修改：修改文档中的指定字段值



在RestClient的API中，全量修改与新增的API完全一致，判断依据是ID：

- 如果新增时，ID已经存在，则修改
- 如果新增时，ID不存在，则新增

这里不再赘述，我们主要关注增量修改。

代码示例如图：

![image-20210720231040875](/home/chanservy/文档/阿面试突击/史上最全微服务/学习资料实用篇/day05-Elasticsearch01/讲义/assets/image-20210720231040875.png)

与之前类似，也是三步走：

- 1）准备Request对象。这次是修改，所以是UpdateRequest
- 2）准备参数。也就是JSON文档，里面包含要修改的字段
- 3）更新文档。这里调用client.update()方法





#### 完整代码

在hotel-demo的HotelDocumentTest测试类中，编写单元测试：

```java
@Test
void testUpdateDocument() throws IOException {
    // 1.准备Request
    UpdateRequest request = new UpdateRequest("hotel", "61083");
    // 2.准备请求参数
    request.doc(
        "price", "952",
        "starName", "四钻"
    );
    // 3.发送请求
    client.update(request, RequestOptions.DEFAULT);
}
```





### 批量导入文档

案例需求：利用BulkRequest批量将数据库数据导入到索引库中。

步骤如下：

- 利用mybatis-plus查询酒店数据

- 将查询到的酒店数据（Hotel）转换为文档类型数据（HotelDoc）

- 利用JavaRestClient中的BulkRequest批处理，实现批量新增文档



#### 语法说明

批量处理BulkRequest，其本质就是将多个普通的CRUD请求组合在一起发送。

其中提供了一个add方法，用来添加其他请求：

![image-20210720232105943](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720232105943.png)

可以看到，能添加的请求包括：

- IndexRequest，也就是新增
- UpdateRequest，也就是修改
- DeleteRequest，也就是删除

因此Bulk中添加了多个IndexRequest，就是批量新增功能了。示例：

![image-20210720232431383](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210720232431383.png)



其实还是三步走：

- 1）创建Request对象。这里是BulkRequest
- 2）准备参数。批处理的参数，就是其它Request对象，这里就是多个IndexRequest
- 3）发起请求。这里是批处理，调用的方法为client.bulk()方法



我们在导入酒店数据时，将上述代码改造成for循环处理即可。

#### 完整代码

在hotel-demo的HotelDocumentTest测试类中，编写单元测试：

```java
@Test
void testBulkRequest() throws IOException {
    // 批量查询酒店数据
    List<Hotel> hotels = hotelService.list();

    // 1.创建Request
    BulkRequest request = new BulkRequest();
    // 2.准备参数，添加多个新增的Request
    for (Hotel hotel : hotels) {
        // 2.1.转换为文档类型HotelDoc
        HotelDoc hotelDoc = new HotelDoc(hotel);
        // 2.2.创建新增文档的Request对象
        request.add(new IndexRequest("hotel")
                    .id(hotelDoc.getId().toString())
                    .source(JSON.toJSONString(hotelDoc), XContentType.JSON));
    }
    // 3.发送请求
    client.bulk(request, RequestOptions.DEFAULT);
}
```

### 小结

文档操作的基本步骤：

- 初始化RestHighLevelClient
- 创建XxxRequest。XXX是Index、Get、Update、Delete、Bulk
- 准备参数（Index、Update、Bulk时需要）
- 发送请求。调用RestHighLevelClient#.xxx()方法，xxx是index、get、update、delete、bulk
- 解析结果（Get时需要）







## DSL查询文档

elasticsearch的查询依然是基于JSON风格的DSL来实现的。

### DSL查询分类

Elasticsearch提供了基于JSON的DSL（[Domain Specific Language](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)）来定义查询。常见的查询类型包括：

- **查询所有**：查询出所有数据，一般测试用。例如：match_all

- **全文检索（full text）查询**：利用分词器对用户输入内容分词，然后去倒排索引库中匹配。例如：
  - match_query
  - multi_match_query
- **精确查询**：根据精确词条值查找数据，一般是查找keyword、数值、日期、boolean等类型字段。例如：
  - ids
  - range
  - term
- **地理（geo）查询**：根据经纬度查询。例如：
  - geo_distance
  - geo_bounding_box
- **复合（compound）查询**：复合查询可以将上述各种查询条件组合起来，合并查询条件。例如：
  - bool
  - function_score

查询的语法基本一致：

```json
GET /indexName/_search
{
  "query": {
    "查询类型": {
      "查询条件": "条件值"
    }
  }
}
```

我们以查询所有为例，其中：

- 查询类型为match_all
- 没有查询条件

```json
// 查询所有
GET /indexName/_search
{
  "query": {
    "match_all": {
    }
  }
}
```

其它查询无非就是**查询类型**、**查询条件**的变化。

### 全文检索查询

#### 使用场景

全文检索查询的基本流程如下：

- 对用户搜索的内容做分词，得到词条
- 根据词条去倒排索引库中匹配，得到文档id
- 根据文档id找到文档，返回给用户

比较常用的场景包括：

- 商城的输入框搜索
- 百度输入框搜索

例如京东：

![image-20210721165326938](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721165326938.png)

因为是拿着词条去匹配，因此参与搜索的字段也必须是可分词的text类型的字段。

#### 基本语法

常见的全文检索查询包括：

- match查询：单字段查询
- multi_match查询：多字段查询，任意一个字段符合条件就算符合查询条件

**match查询语法如下：**

```json
GET /indexName/_search
{
  "query": {
    "match": {
      "FIELD": "TEXT"
    }
  }
}
```

or关系：`match`类型查询，会把查询条件进行分词，然后进行查询，多个词条之间默认是or的关系，或的意思，表示满足一个条件就行。

某些情况下，我们需要更精确查找，我们希望这个关系变成`and`，所有条件都要满足，可以这样做：

```json
GET /hotel/_search
{
    "query":{
        "match": {
          "title": {
            "query": "外滩如家",
            "operator": "and"
          }
        }
    }
}
```

本例中，只有同时包含`外滩`和`如家`的词条才会被搜索到。

> or和and之间？

在 `or` 与 `and` 间二选一有点过于非黑即白。 如果用户给定的条件分词后有 5 个查询词项，想查找只包含其中 4 个词的文档，该如何处理？将 operator 操作符参数设置成 `and` 只会将此文档排除。

有时候这正是我们期望的，但在全文搜索的大多数应用场景下，我们既想包含那些可能相关的文档，同时又排除那些不太相关的。换句话说，我们想要处于中间某种结果。

`match` 查询支持 `minimum_should_match` 最小匹配参数， 这让我们可以指定必须匹配的词项数用来表示一个文档是否相关。我们可以将其设置为某个具体数字，更常用的做法是将其设置为一个`百分数`，因为我们无法控制用户搜索时输入的单词数量：

```json
GET /hotel/_search
{
    "query":{
        "match":{
            "title":{
              "query":"外滩如家酒店",
              "minimum_should_match": "75%"
            }
        }
    }
}
```

本例中，搜索语句可以分为3个词，如果使用and关系，需要同时满足3个词才会被搜索到。这里我们采用最小匹配参数：75%，那么也就是说只要匹配到总词条数量的75%即可，这里3*75% 约等于2。所以只要包含2个词条就算满足条件了。

**mulit_match语法如下：**

```json
GET /indexName/_search
{
  "query": {
    "multi_match": {
      "query": "TEXT",
      "fields": ["FIELD1", " FIELD12"]
    }
  }
}
```

#### 示例

match查询示例：

![image-20210721170455419](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721170455419.png)



multi_match查询示例：

![image-20210721170720691](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721170720691.png)



可以看到，两种查询结果是一样的，为什么？

因为我们将brand、name、business值都利用copy_to复制到了all字段中。因此你根据三个字段搜索，和根据all字段搜索效果当然一样了。

但是，搜索字段越多，对查询性能影响越大，因此建议采用copy_to，然后单字段查询的方式。

#### 总结

match和multi_match的区别是什么？

- match：根据一个字段查询
- multi_match：根据多个字段查询，参与查询字段越多，查询性能越差

### 精准查询

精确查询一般是查找keyword、数值、日期、boolean等类型字段。所以**不会**对搜索条件分词。常见的有：

- term：根据词条精确值查询
- range：根据值的范围查询

#### term查询

因为精确查询的字段搜是不分词的字段，因此查询的条件也必须是**不分词**的词条。查询时，用户输入的内容跟自动值完全匹配时才认为符合条件。如果用户输入的内容过多，反而搜索不到数据。

语法说明：

```json
// term查询
GET /indexName/_search
{
  "query": {
    "term": {
      "FIELD": {
        "value": "VALUE"
      }
    }
  }
}
```

示例：

当我搜索的是精确词条时，能正确查询出结果：

![image-20210721171655308](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721171655308.png)

但是，当我搜索的内容不是词条，而是多个词语形成的短语时，反而搜索不到：

![image-20210721171838378](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721171838378.png)



#### terms查询（多词条）

`terms` 查询和 term 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件。

#### range查询

范围查询，一般应用在对数值类型做范围过滤的时候。比如做价格范围过滤。

基本语法：

```json
// range查询
GET /indexName/_search
{
  "query": {
    "range": {
      "FIELD": {
        "gte": 10, // 这里的gte代表大于等于，gt则代表大于
        "lte": 20 // lte代表小于等于，lt则代表小于
      }
    }
  }
}
```

`range`查询允许以下字符：

| 操作符 |   说明   |
| :----: | :------: |
|   gt   |   大于   |
|  gte   | 大于等于 |
|   lt   |   小于   |
|  lte   | 小于等于 |

示例：

![image-20210721172307172](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721172307172.png)

#### 总结

精确查询常见的有哪些？

- term查询：根据词条精确匹配，一般搜索keyword类型、数值类型、布尔类型、日期类型字段
- range查询：根据数值范围查询，可以是数值、日期的范围



### 结果过滤

默认情况下，elasticsearch在搜索的结果中，会把文档中保存在`_source`的所有字段都返回。

如果我们只想获取其中的部分字段，我们可以添加`_source`的过滤，属于一种优化，对不用的字段不做查询。

#### 直接指定字段

示例：

```json
GET /hotel/_search
{
  "_source": ["title","price"],
  "query": {
    "term": {
      "price": 2699
    }
  }
}
```

#### 指定includes和excludes

我们也可以通过：

- includes：来指定想要显示的字段
- excludes：来指定不想要显示的字段

二者都是可选的。

示例：

```json
GET /hotel/_search
{
  "_source": {
    "includes":["title","price"]
  },
  "query": {
    "term": {
      "price": 2699
    }
  }
}
```

如果只有title，price，images这三个属性，那么上面与下面的结果将是一样的：

```json
GET /hotel/_search
{
  "_source": {
     "excludes": ["images"]
  },
  "query": {
    "term": {
      "price": 2699
    }
  }
}
```



### 地理坐标查询

所谓的地理坐标查询，其实就是根据经纬度查询，官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-queries.html

常见的使用场景包括：

- 携程：搜索我附近的酒店
- 滴滴：搜索我附近的出租车
- 微信：搜索我附近的人

附近的酒店：

![image-20210721172645103](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721172645103.png) 

附近的车：

![image-20210721172654880](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721172654880.png) 



#### 矩形范围查询

矩形范围查询，也就是geo_bounding_box查询，查询坐标落在某个矩形范围的所有文档：

![DKV9HZbVS6](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/DKV9HZbVS6.gif)

查询时，需要指定矩形的**左上**、**右下**两个点的坐标，然后画出一个矩形，落在该矩形内的都是符合条件的点。

语法如下：

```json
// geo_bounding_box查询
GET /indexName/_search
{
  "query": {
    "geo_bounding_box": {
      "FIELD": {
        "top_left": { // 左上点
          "lat": 31.1,
          "lon": 121.5
        },
        "bottom_right": { // 右下点
          "lat": 30.9,
          "lon": 121.7
        }
      }
    }
  }
}
```

这种并不符合“附近的人”这样的需求，所以我们就不做了。



#### 附近查询

附近查询，也叫做距离查询（geo_distance）：查询到指定中心点小于某个距离值的所有文档。

换句话来说，在地图上找一个点作为圆心，以指定距离为半径，画一个圆，落在圆内的坐标都算符合条件：

![vZrdKAh19C](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/vZrdKAh19C.gif)

语法说明：

```json
// geo_distance 查询
GET /indexName/_search
{
  "query": {
    "geo_distance": {
      "distance": "15km", // 半径
      "FIELD": "31.21,121.5" // 圆心
    }
  }
}
```

示例：

我们先搜索陆家嘴附近15km的酒店：

![image-20210721175443234](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721175443234.png)

发现共有47家酒店。

然后把半径缩短到3公里：

![image-20210721182031475](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721182031475.png)

可以发现，搜索到的酒店数量减少到了5家。



### 过滤（filter）

> 条件查询中进行过滤

所有的查询都会影响到文档的评分及排名。如果我们需要在查询结果中进行过滤，并且不希望过滤条件影响评分，那么就不要把过滤条件作为查询条件来用。而是使用`filter`方式，也就是说将范围查询写到filter中。其实范围查询就相当于过滤啦。

```json
GET /csy2/_search
{
    "query":{
        "bool":{
        	"must":{ "match": { "title": "小米手机" }},
        	"filter":{
                "range":{"price":{"gt":2000.00,"lt":3800.00}}
        	}
        }
    }
}
```

注意：`filter`中还可以再次进行`bool`组合条件过滤。

> 无查询条件，直接过滤

如果一次查询只有过滤，没有查询条件，不希望进行评分，我们可以使用`constant_score`取代只有 filter 语句的 bool 查询。在性能上是完全相同的，但对于提高查询简洁性和清晰度有很大帮助。

```json
GET /csy2/_search
{
    "query":{
        "constant_score":{
        	"filter":{
                "range":{"price":{"gt":2000.00,"lt":3800.00}}
        	}
        }
    }
}
```



### 复合查询

复合（compound）查询：复合查询可以将其它简单查询组合起来，实现更复杂的搜索逻辑。常见的有两种：

- fuction score：算分函数查询，可以控制文档相关性算分，控制文档排名
- bool query：布尔查询，利用逻辑关系组合多个其它的查询，实现复杂搜索



#### 相关性算分

当我们利用match查询时，文档结果会根据与搜索词条的关联度打分（_score），返回结果时按照分值降序排列。

例如，我们搜索 "虹桥如家"，结果如下：

```json
[
  {
    "_score" : 17.850193,
    "_source" : {
      "name" : "虹桥如家酒店真不错",
    }
  },
  {
    "_score" : 12.259849,
    "_source" : {
      "name" : "外滩如家酒店真不错",
    }
  },
  {
    "_score" : 11.91091,
    "_source" : {
      "name" : "迪士尼如家酒店真不错",
    }
  }
]
```



在elasticsearch中，早期使用的打分算法是TF-IDF算法，公式如下：

![image-20210721190152134](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721190152134.png)



在后来的5.1版本升级中，elasticsearch将算法改进为BM25算法，公式如下：

![image-20210721190416214](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721190416214.png)





TF-IDF算法有一各缺陷，就是词条频率越高，文档得分也会越高，单个词条对文档影响较大。而BM25则会让单个词条的算分有一个上限，曲线更加平滑：

![image-20210721190907320](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721190907320.png)



小结：elasticsearch会根据词条和文档的相关度做打分，算法由两种：

- TF-IDF算法
- BM25算法，elasticsearch5.1版本后采用的算法



#### 算分函数查询

根据相关度打分是比较合理的需求，但**合理的不一定是产品经理需要**的。

以百度为例，你搜索的结果中，并不是相关度越高排名越靠前，而是谁掏的钱多排名就越靠前。如图：

![image-20210721191144560](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721191144560.png)



要想认为控制相关性算分，就需要利用elasticsearch中的function score 查询了。



##### 1）语法说明

![image-20210721191544750](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721191544750.png)



function score 查询中包含四部分内容：

- **原始查询**条件：query部分，基于这个条件搜索文档，并且基于BM25算法给文档打分，**原始算分**（query score)
- **过滤条件**：filter部分，符合该条件的文档才会重新算分
- **算分函数**：符合filter条件的文档要根据这个函数做运算，得到的**函数算分**（function score），有四种函数
  - weight：函数结果是常量
  - field_value_factor：以文档中的某个字段值作为函数结果
  - random_score：以随机数作为函数结果
  - script_score：自定义算分函数算法
- **运算模式**：算分函数的结果、原始查询的相关性算分，两者之间的运算方式，包括：
  - multiply：相乘
  - replace：用function score替换query score
  - 其它，例如：sum、avg、max、min



function score的运行流程如下：

- 1）根据**原始条件**查询搜索文档，并且计算相关性算分，称为**原始算分**（query score）
- 2）根据**过滤条件**，过滤文档
- 3）符合**过滤条件**的文档，基于**算分函数**运算，得到**函数算分**（function score）
- 4）将**原始算分**（query score）和**函数算分**（function score）基于**运算模式**做运算，得到最终结果，作为相关性算分。



因此，其中的关键点是：

- 过滤条件：决定哪些文档的算分被修改
- 算分函数：决定函数算分的算法
- 运算模式：决定最终算分结果



##### 2）示例

需求：给“如家”这个品牌的酒店排名靠前一些

翻译一下这个需求，转换为之前说的四个要点：

- 原始条件：不确定，可以任意变化
- 过滤条件：brand = "如家"
- 算分函数：可以简单粗暴，直接给固定的算分结果，weight
- 运算模式：比如求和

因此最终的DSL语句如下：

```json
GET /hotel/_search
{
  "query": {
    "function_score": {
      "query": {  .... }, // 原始查询，可以是任意条件
      "functions": [ // 算分函数
        {
          "filter": { // 满足的条件，品牌必须是如家
            "term": {
              "brand": "如家"
            }
          },
          "weight": 2 // 算分权重为2
        }
      ],
      "boost_mode": "sum" // 加权模式，求和
    }
  }
}
```



测试，在未添加算分函数时，如家得分如下：

![image-20210721193152520](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721193152520.png)

添加了算分函数后，如家得分就提升了：

![image-20210721193458182](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721193458182.png)



##### 3）小结

function score query定义的三要素是什么？

- 过滤条件：哪些文档要加分
- 算分函数：如何计算function score
- 加权方式：function score 与 query score如何运算



#### 布尔查询

布尔查询是一个或多个查询子句的组合，每一个子句就是一个**子查询**。子查询的组合方式有：

- must：必须匹配每个子查询，类似“与”
- should：选择性匹配子查询，类似“或”
- must_not：必须不匹配，**不参与算分**，类似“非”
- filter：必须匹配，**不参与算分**

`bool`把各种其它查询通过`must`（与）、`must_not`（非）、`should`（或）的方式进行组合。

比如在搜索酒店时，除了关键字搜索外，我们还可能根据品牌、价格、城市等字段做过滤：

![image-20210721193822848](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721193822848.png)

每一个不同的字段，其查询的条件、方式都不一样，必须是多个不同的查询，而要组合这些查询，就必须用bool查询了。



需要注意的是，搜索时，参与**打分的字段越多，查询的性能也越差**。因此这种多条件查询时，建议这样做：

- 搜索框的关键字搜索，是全文检索查询，使用must查询，参与算分
- 其它过滤条件，采用filter查询。不参与算分



##### 1）语法示例

```json
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"city": "上海" }}
      ],
      "should": [
        {"term": {"brand": "皇冠假日" }},
        {"term": {"brand": "华美达" }}
      ],
      "must_not": [
        { "range": { "price": { "lte": 500 } }}
      ],
      "filter": [
        { "range": {"score": { "gte": 45 } }}
      ]
    }
  }
}
```



##### 2）示例

需求：搜索名字包含“如家”，价格不高于400，在坐标31.21,121.5周围10km范围内的酒店。

分析：

- 名称搜索，属于全文检索查询，应该参与算分。放到must中
- 价格不高于400，用range查询，属于过滤条件，不参与算分。放到must_not中
- 周围10km范围内，用geo_distance查询，属于过滤条件，不参与算分。放到filter中



![image-20210721194744183](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721194744183.png)





##### 3）小结

bool查询有几种逻辑关系？

- must：必须匹配的条件，可以理解为“与”
- should：选择性匹配的条件，可以理解为“或”
- must_not：必须不匹配的条件，不参与打分
- filter：必须匹配的条件，不参与打分



## 搜索结果处理

搜索的结果可以按照用户指定的方式去处理或展示。

### 排序

elasticsearch默认是根据相关度算分（_score）来排序，但是也支持自定义方式对搜索[结果排序](https://www.elastic.co/guide/en/elasticsearch/reference/current/sort-search-results.html)。可以排序字段类型有：keyword类型、数值类型、地理坐标类型、日期类型等。

#### 普通字段排序

keyword、数值、日期类型排序的语法基本一致。

**语法**：

```json
GET /indexName/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "FIELD": "desc"  // 排序字段、排序方式ASC、DESC
    }
  ]
}
```

排序条件是一个数组，也就是可以写多个排序条件。按照声明的顺序，当第一个条件相等时，再按照第二个条件排序，以此类推



**示例**：

需求描述：酒店数据按照用户评价（score)降序排序，评价相同的按照价格(price)升序排序

![image-20210721195728306](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721195728306.png)



#### 地理坐标排序

地理坐标排序略有不同。

**语法说明**：

```json
GET /indexName/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_geo_distance" : {
          "FIELD" : "纬度，经度", // 文档中geo_point类型的字段名、目标坐标点
          "order" : "asc", // 排序方式
          "unit" : "km" // 排序的距离单位
      }
    }
  ]
}
```

这个查询的含义是：

- 指定一个坐标，作为目标点
- 计算每一个文档中，指定字段（必须是geo_point类型）的坐标 到目标点的距离是多少
- 根据距离排序



**示例：**

需求描述：实现对酒店数据按照到你的位置坐标的距离升序排序

提示：获取你的位置的经纬度的方式：https://lbs.amap.com/demo/jsapi-v2/example/map/click-to-get-lnglat/



假设我的位置是：31.034661，121.612282，寻找我周围距离最近的酒店。

![image-20210721200214690](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721200214690.png)





### 分页

elasticsearch 默认情况下只返回top10的数据。而如果要查询更多数据就需要修改分页参数了。elasticsearch中通过修改from、size参数来控制要返回的分页结果：

- from：从第几个文档开始
- size：总共查询几个文档

类似于mysql中的`limit ?, ?`

#### 基本的分页

分页的基本语法如下：

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0, // 分页开始的位置，默认为0
  "size": 10, // 期望获取的文档总数
  "sort": [
    {"price": "asc"}
  ]
}
```





#### 深度分页问题

现在，我要查询990~1000的数据，查询逻辑要这么写：

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "from": 990, // 分页开始的位置，默认为0
  "size": 10, // 期望获取的文档总数
  "sort": [
    {"price": "asc"}
  ]
}
```

这里是查询990开始的数据，也就是 第990~第1000条 数据。

不过，elasticsearch内部分页时，必须先查询 0~1000条，然后截取其中的990 ~ 1000的这10条：

![image-20210721200643029](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721200643029.png)



查询TOP1000，如果es是单点模式，这并无太大影响。

但是elasticsearch将来一定是集群，例如我集群有5个节点，我要查询TOP1000的数据，并不是每个节点查询200条就可以了。

因为节点A的TOP200，在另一个节点可能排到10000名以外了。

因此要想获取整个集群的TOP1000，必须先查询出每个节点的TOP1000，汇总结果后，重新排名，重新截取TOP1000。

![image-20210721201003229](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721201003229.png)



那如果我要查询9900~10000的数据呢？是不是要先查询TOP10000呢？那每个节点都要查询10000条？汇总到内存中？

当查询分页深度较大时，汇总数据过多，对内存和CPU会产生非常大的压力，因此elasticsearch会禁止from+ size 超过10000的请求。

针对深度分页，ES提供了两种解决方案，[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html)：

- search after：分页时需要排序，原理是从上一次的排序值开始，查询下一页数据。官方推荐使用的方式。
- scroll：原理将排序后的文档id形成快照，保存在内存。官方已经不推荐使用。



#### 小结

分页查询的常见实现方案以及优缺点：

- `from + size`：
  - 优点：支持随机翻页
  - 缺点：深度分页问题，默认查询上限（from + size）是10000
  - 场景：百度、京东、谷歌、淘宝这样的随机翻页搜索
- `after search`：
  - 优点：没有查询上限（单次查询的size不超过10000）
  - 缺点：只能向后逐页查询，不支持随机翻页
  - 场景：没有随机翻页需求的搜索，例如手机向下滚动翻页

- `scroll`：
  - 优点：没有查询上限（单次查询的size不超过10000）
  - 缺点：会有额外内存消耗，并且搜索结果是非实时的
  - 场景：海量数据的获取和迁移。从ES7.1开始不推荐，建议用 after search方案。





### 高亮

#### 高亮原理

什么是高亮显示呢？

我们在百度，京东搜索时，关键字会变成红色，比较醒目，这叫高亮显示：

![image-20210721202705030](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721202705030.png)

高亮显示的实现分为两步：

- 1）给文档中的所有关键字都添加一个标签，例如`<em>`标签
- 2）页面给`<em>`标签编写CSS样式



#### 实现高亮

**高亮的语法**：

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "FIELD": "TEXT" // 查询条件，高亮一定要使用全文检索查询
    }
  },
  "highlight": {
    "fields": { // 指定要高亮的字段
      "FIELD": {
        "pre_tags": "<em>",  // 用来标记高亮字段的前置标签
        "post_tags": "</em>" // 用来标记高亮字段的后置标签
      }
    }
  }
}
```



**注意：**

- 高亮是对关键字高亮，因此**搜索条件必须带有关键字**，而不能是范围这样的查询。
- 默认情况下，**高亮的字段，必须与搜索指定的字段一致**，否则无法高亮
- 如果要对非搜索字段高亮，则需要添加一个属性：required_field_match=false



**示例**：

![image-20210721203349633](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721203349633.png)





### 总结

查询的DSL是一个大的JSON对象，包含下列属性：

- query：查询条件
- from和size：分页条件
- sort：排序条件
- highlight：高亮条件

示例：

![image-20210721203657850](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/image-20210721203657850.png)





## RestClient之DSL查询文档

文档的查询同样适用于 RestHighLevelClient 对象，基本步骤包括：

- 1）准备Request对象
- 2）准备请求参数
- 3）发起请求
- 4）解析响应



### 快速入门

我们以match_all查询为例

#### 发起查询请求

![image-20210721203950559](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721203950559.png)

代码解读：

- 第一步，创建`SearchRequest`对象，指定索引库名

- 第二步，利用`request.source()`构建DSL，DSL中可以包含查询、分页、排序、高亮等
  - `query()`：代表查询条件，利用`QueryBuilders.matchAllQuery()`构建一个match_all查询的DSL
- 第三步，利用client.search()发送请求，得到响应



这里关键的API有两个，一个是`request.source()`，其中包含了查询、排序、分页、高亮等所有功能：

![image-20210721215640790](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721215640790.png)

另一个是`QueryBuilders`，其中包含match、term、function_score、bool等各种查询：

![image-20210721215729236](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721215729236.png)



#### 解析响应

响应结果的解析：

![image-20210721214221057](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721214221057.png)



elasticsearch返回的结果是一个JSON字符串，结构包含：

- `hits`：命中的结果
  - `total`：总条数，其中的value是具体的总条数值
  - `max_score`：所有结果中得分最高的文档的相关性算分
  - `hits`：搜索结果的文档数组，其中的每个文档都是一个json对象
    - `_source`：文档中的原始数据，也是json对象

因此，我们解析响应结果，就是逐层解析JSON字符串，流程如下：

- `SearchHits`：通过response.getHits()获取，就是JSON中的最外层的hits，代表命中的结果
  - `SearchHits#getTotalHits().value`：获取总条数信息
  - `SearchHits#getHits()`：获取SearchHit数组，也就是文档数组
    - `SearchHit#getSourceAsString()`：获取文档结果中的_source，也就是原始的json文档数据



#### 完整代码

完整代码如下：

```java
@Test
void testMatchAll() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    request.source()
        .query(QueryBuilders.matchAllQuery());
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);

    // 4.解析响应
    handleResponse(response);
}

private void handleResponse(SearchResponse response) {
    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获取总条数
    long total = searchHits.getTotalHits().value;
    System.out.println("共搜索到" + total + "条数据");
    // 4.2.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    for (SearchHit hit : hits) {
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
        System.out.println("hotelDoc = " + hotelDoc);
    }
}
```



#### 小结

查询的基本步骤是：

1. 创建SearchRequest对象

2. 准备Request.source()，也就是DSL。

   ① QueryBuilders来构建查询条件

   ② 传入Request.source() 的 query() 方法

3. 发送请求，得到结果

4. 解析结果（参考JSON结果，从外到内，逐层解析）





### match查询

全文检索的match和multi_match查询与match_all的API基本一致。差别是查询条件，也就是query的部分。

![image-20210721215923060](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721215923060.png) 



因此，Java代码上的差异主要是request.source().query()中的参数了。同样是利用QueryBuilders提供的方法：

![image-20210721215843099](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721215843099.png) 

而结果解析代码则完全一致，可以抽取并共享。



完整代码如下：

```java
@Test
void testMatch() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    request.source()
        .query(QueryBuilders.matchQuery("all", "如家"));
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);

}
```





### 精确查询

精确查询主要是两者：

- term：词条精确匹配
- range：范围查询

与之前的查询相比，差异同样在查询条件，其它都一样。

查询条件构造的API如下：

![image-20210721220305140](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721220305140.png) 





### 布尔查询

布尔查询是用must、must_not、filter等方式组合其它查询，代码示例如下：

![image-20210721220927286](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721220927286.png)



可以看到，API与其它查询的差别同样是在查询条件的构建，QueryBuilders，结果解析等其他代码完全不变。



完整代码如下：

```java
@Test
void testBool() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    // 2.1.准备BooleanQuery
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    // 2.2.添加term
    boolQuery.must(QueryBuilders.termQuery("city", "杭州"));
    // 2.3.添加range
    boolQuery.filter(QueryBuilders.rangeQuery("price").lte(250));

    request.source().query(boolQuery);
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);

}
```



### 排序、分页

搜索结果的排序和分页是与query同级的参数，因此同样是使用request.source()来设置。

对应的API如下：

![image-20210721221121266](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721221121266.png)



完整代码示例：

```java
@Test
void testPageAndSort() throws IOException {
    // 页码，每页大小
    int page = 1, size = 5;

    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    // 2.1.query
    request.source().query(QueryBuilders.matchAllQuery());
    // 2.2.排序 sort
    request.source().sort("price", SortOrder.ASC);
    // 2.3.分页 from、size
    request.source().from((page - 1) * size).size(5);
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);

}
```



### 高亮

高亮的代码与之前代码差异较大，有两点：

- 查询的DSL：其中除了查询条件，还需要添加高亮条件，同样是与query同级。
- 结果解析：结果除了要解析_source文档数据，还要解析高亮结果

#### 高亮请求构建

高亮请求的构建API如下：

![image-20210721221744883](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721221744883.png)

上述代码省略了查询条件部分，但是大家不要忘了：高亮查询必须使用全文检索查询，并且要有搜索关键字，将来才可以对关键字高亮。

完整代码如下：

```java
@Test
void testHighlight() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    // 2.1.query
    request.source().query(QueryBuilders.matchQuery("all", "如家"));
    // 2.2.高亮
    request.source().highlighter(new HighlightBuilder().field("name").requireFieldMatch(false));
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);

}
```



#### 高亮结果解析

高亮的结果与查询的文档结果默认是分离的，并不在一起。

因此解析高亮的代码需要额外处理：

![image-20210721222057212](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721222057212.png)

代码解读：

- 第一步：从结果中获取source。hit.getSourceAsString()，这部分是非高亮结果，json字符串。还需要反序列为HotelDoc对象
- 第二步：获取高亮结果。hit.getHighlightFields()，返回值是一个Map，key是高亮字段名称，值是HighlightField对象，代表高亮值
- 第三步：从map中根据高亮字段名称，获取高亮字段值对象HighlightField
- 第四步：从HighlightField中获取Fragments，并且转为字符串。这部分就是真正的高亮字符串了
- 第五步：用高亮的结果替换HotelDoc中的非高亮结果



完整代码如下：

```java
private void handleResponse(SearchResponse response) {
    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获取总条数
    long total = searchHits.getTotalHits().value;
    System.out.println("共搜索到" + total + "条数据");
    // 4.2.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    for (SearchHit hit : hits) {
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
        // 获取高亮结果
        Map<String, HighlightField> highlightFields = hit.getHighlightFields();
        if (!CollectionUtils.isEmpty(highlightFields)) {
            // 根据字段名获取高亮结果
            HighlightField highlightField = highlightFields.get("name");
            if (highlightField != null) {
                // 获取高亮值
                String name = highlightField.getFragments()[0].string();
                // 覆盖非高亮结果
                hotelDoc.setName(name);
            }
        }
        System.out.println("hotelDoc = " + hotelDoc);
    }
}
```





## 黑马旅游案例

下面，我们通过黑马旅游的案例来实战演练下之前学习的知识。

我们实现四部分功能：

- 酒店搜索和分页
- 酒店结果过滤
- 我周边的酒店
- 酒店竞价排名



启动我们提供的hotel-demo项目，其默认端口是8089，访问http://localhost:8090，就能看到项目页面了：

![image-20210721223159598](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721223159598.png)





### 酒店搜索和分页

案例需求：实现黑马旅游的酒店搜索功能，完成关键字搜索和分页

#### 需求分析

在项目的首页，有一个大大的搜索框，还有分页按钮：

![image-20210721223859419](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721223859419.png)

点击搜索按钮，可以看到浏览器控制台发出了请求：

![image-20210721224033789](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721224033789.png)

请求参数如下：

![image-20210721224112708](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721224112708.png)



由此可以知道，我们这个请求的信息如下：

- 请求方式：POST
- 请求路径：/hotel/list
- 请求参数：JSON对象，包含4个字段：
  - key：搜索关键字
  - page：页码
  - size：每页大小
  - sortBy：排序，目前暂不实现
- 返回值：分页查询，需要返回分页结果PageResult，包含两个属性：
  - `total`：总条数
  - `List<HotelDoc>`：当前页的数据



因此，我们实现业务的流程如下：

- 步骤一：定义实体类，接收请求参数的JSON对象
- 步骤二：编写controller，接收页面的请求
- 步骤三：编写业务实现，利用RestHighLevelClient实现搜索、分页



#### 定义实体类

实体类有两个，一个是前端的请求参数实体，一个是服务端应该返回的响应结果实体。

1）请求参数

前端请求的json结构如下：

```json
{
    "key": "搜索关键字",
    "page": 1,
    "size": 3,
    "sortBy": "default"
}
```

因此，我们在`cn.itcast.hotel.pojo`包下定义一个实体类：

```java
package cn.itcast.hotel.pojo;

import lombok.Data;

@Data
public class RequestParams {
    private String key;
    private Integer page;
    private Integer size;
    private String sortBy;
}
```



2）返回值

分页查询，需要返回分页结果PageResult，包含两个属性：

- `total`：总条数
- `List<HotelDoc>`：当前页的数据

因此，我们在`cn.itcast.hotel.pojo`中定义返回结果：

```java
package cn.itcast.hotel.pojo;

import lombok.Data;

import java.util.List;

@Data
public class PageResult {
    private Long total;
    private List<HotelDoc> hotels;

    public PageResult() {
    }

    public PageResult(Long total, List<HotelDoc> hotels) {
        this.total = total;
        this.hotels = hotels;
    }
}
```



#### 定义controller

定义一个HotelController，声明查询接口，满足下列要求：

- 请求方式：Post
- 请求路径：/hotel/list
- 请求参数：对象，类型为RequestParam
- 返回值：PageResult，包含两个属性
  - `Long total`：总条数
  - `List<HotelDoc> hotels`：酒店数据



因此，我们在`cn.itcast.hotel.web`中定义HotelController：

```java
@RestController
@RequestMapping("/hotel")
public class HotelController {

    @Autowired
    private IHotelService hotelService;
	// 搜索酒店数据
    @PostMapping("/list")
    public PageResult search(@RequestBody RequestParams params){
        return hotelService.search(params);
    }
}
```



#### 实现搜索业务

我们在controller调用了IHotelService，并没有实现该方法，因此下面我们就在IHotelService中定义方法，并且去实现业务逻辑。

1）在`cn.itcast.hotel.service`中的`IHotelService`接口中定义一个方法：

```java
/**
 * 根据关键字搜索酒店信息
 * @param params 请求参数对象，包含用户输入的关键字 
 * @return 酒店文档列表
 */
PageResult search(RequestParams params);
```



2）实现搜索业务，肯定离不开RestHighLevelClient，我们需要把它注册到Spring中作为一个Bean。在`cn.itcast.hotel`中的`HotelDemoApplication`中声明这个Bean：

```java
@Bean
public RestHighLevelClient client(){
    return  new RestHighLevelClient(RestClient.builder(
        HttpHost.create("http://192.168.150.101:9200")
    ));
}
```





3）在`cn.itcast.hotel.service.impl`中的`HotelService`中实现search方法：

```java
@Override
public PageResult search(RequestParams params) {
    try {
        // 1.准备Request
        SearchRequest request = new SearchRequest("hotel");
        // 2.准备DSL
        // 2.1.query
        String key = params.getKey();
        if (key == null || "".equals(key)) {
            boolQuery.must(QueryBuilders.matchAllQuery());
        } else {
            boolQuery.must(QueryBuilders.matchQuery("all", key));
        }

        // 2.2.分页
        int page = params.getPage();
        int size = params.getSize();
        request.source().from((page - 1) * size).size(size);

        // 3.发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 4.解析响应
        return handleResponse(response);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}

// 结果解析
private PageResult handleResponse(SearchResponse response) {
    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获取总条数
    long total = searchHits.getTotalHits().value;
    // 4.2.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    List<HotelDoc> hotels = new ArrayList<>();
    for (SearchHit hit : hits) {
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
		// 放入集合
        hotels.add(hotelDoc);
    }
    // 4.4.封装返回
    return new PageResult(total, hotels);
}
```





### 酒店结果过滤

需求：添加品牌、城市、星级、价格等过滤功能

#### 需求分析

在页面搜索框下面，会有一些过滤项：

![image-20210722091940726](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722091940726.png)

传递的参数如图：

![image-20210722092051994](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722092051994.png) 

包含的过滤条件有：

- brand：品牌值
- city：城市
- minPrice~maxPrice：价格范围
- starName：星级

我们需要做两件事情：

- 修改请求参数的对象RequestParams，接收上述参数
- 修改业务逻辑，在搜索条件之外，添加一些过滤条件



#### 修改实体类

修改在`cn.itcast.hotel.pojo`包下的实体类RequestParams：

```java
@Data
public class RequestParams {
    private String key;
    private Integer page;
    private Integer size;
    private String sortBy;
    // 下面是新增的过滤条件参数
    private String city;
    private String brand;
    private String starName;
    private Integer minPrice;
    private Integer maxPrice;
}
```



#### 修改搜索业务

在HotelService的search方法中，只有一个地方需要修改：requet.source().query( ... )其中的查询条件。

在之前的业务中，只有match查询，根据关键字搜索，现在要添加条件过滤，包括：

- 品牌过滤：是keyword类型，用term查询
- 星级过滤：是keyword类型，用term查询
- 价格过滤：是数值类型，用range查询
- 城市过滤：是keyword类型，用term查询

多个查询条件组合，肯定是boolean查询来组合：

- 关键字搜索放到must中，参与算分
- 其它过滤条件放到filter中，不参与算分



因为条件构建的逻辑比较复杂，这里先封装为一个函数：

![image-20210722092935453](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722092935453.png)



buildBasicQuery的代码如下：

```java
private void buildBasicQuery(RequestParams params, SearchRequest request) {
    // 1.构建BooleanQuery
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    // 2.关键字搜索
    String key = params.getKey();
    if (key == null || "".equals(key)) {
        boolQuery.must(QueryBuilders.matchAllQuery());
    } else {
        boolQuery.must(QueryBuilders.matchQuery("all", key));
    }
    // 3.城市条件
    if (params.getCity() != null && !params.getCity().equals("")) {
        boolQuery.filter(QueryBuilders.termQuery("city", params.getCity()));
    }
    // 4.品牌条件
    if (params.getBrand() != null && !params.getBrand().equals("")) {
        boolQuery.filter(QueryBuilders.termQuery("brand", params.getBrand()));
    }
    // 5.星级条件
    if (params.getStarName() != null && !params.getStarName().equals("")) {
        boolQuery.filter(QueryBuilders.termQuery("starName", params.getStarName()));
    }
	// 6.价格
    if (params.getMinPrice() != null && params.getMaxPrice() != null) {
        boolQuery.filter(QueryBuilders
                         .rangeQuery("price")
                         .gte(params.getMinPrice())
                         .lte(params.getMaxPrice())
                        );
    }
	// 7.放入source
    request.source().query(boolQuery);
}
```





### 我周边的酒店

需求：我附近的酒店

#### 需求分析

在酒店列表页的右侧，有一个小地图，点击地图的定位按钮，地图会找到你所在的位置：

![image-20210722093414542](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722093414542.png) 

并且，在前端会发起查询请求，将你的坐标发送到服务端：

![image-20210722093642382](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722093642382.png) 



我们要做的事情就是基于这个location坐标，然后按照距离对周围酒店排序。实现思路如下：

- 修改RequestParams参数，接收location字段
- 修改search方法业务逻辑，如果location有值，添加根据geo_distance排序的功能



#### 修改实体类

修改在`cn.itcast.hotel.pojo`包下的实体类RequestParams：

```java
package cn.itcast.hotel.pojo;

import lombok.Data;

@Data
public class RequestParams {
    private String key;
    private Integer page;
    private Integer size;
    private String sortBy;
    private String city;
    private String brand;
    private String starName;
    private Integer minPrice;
    private Integer maxPrice;
    // 我当前的地理坐标
    private String location;
}

```



#### 距离排序API

我们以前学习过排序功能，包括两种：

- 普通字段排序
- 地理坐标排序

我们只讲了普通字段排序对应的java写法。地理坐标排序只学过DSL语法，如下：

```json
GET /indexName/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": "asc"  
    },
    {
      "_geo_distance" : {
          "FIELD" : "纬度，经度",
          "order" : "asc",
          "unit" : "km"
      }
    }
  ]
}
```

对应的java代码示例：

![image-20210722095227059](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722095227059.png)





#### 添加距离排序

在`cn.itcast.hotel.service.impl`的`HotelService`的`search`方法中，添加一个排序功能：

![image-20210722095902314](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722095902314.png)



完整代码：

```java
@Override
public PageResult search(RequestParams params) {
    try {
        // 1.准备Request
        SearchRequest request = new SearchRequest("hotel");
        // 2.准备DSL
        // 2.1.query
        buildBasicQuery(params, request);

        // 2.2.分页
        int page = params.getPage();
        int size = params.getSize();
        request.source().from((page - 1) * size).size(size);

        // 2.3.排序
        String location = params.getLocation();
        if (location != null && !location.equals("")) {
            request.source().sort(SortBuilders
                                  .geoDistanceSort("location", new GeoPoint(location))
                                  .order(SortOrder.ASC)
                                  .unit(DistanceUnit.KILOMETERS)
                                 );
        }

        // 3.发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 4.解析响应
        return handleResponse(response);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```



#### 排序距离显示

重启服务后，测试我的酒店功能：

![image-20210722100040674](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722100040674.png)



发现确实可以实现对我附近酒店的排序，不过并没有看到酒店到底距离我多远，这该怎么办？



排序完成后，页面还要获取我附近每个酒店的具体**距离**值，这个值在响应结果中是独立的：

![image-20210722095648542](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722095648542.png)

因此，我们在结果解析阶段，除了解析source部分以外，还要得到sort部分，也就是排序的距离，然后放到响应结果中。

我们要做两件事：

- 修改HotelDoc，添加排序距离字段，用于页面显示
- 修改HotelService类中的handleResponse方法，添加对sort值的获取



1）修改HotelDoc类，添加距离字段

```java
package cn.itcast.hotel.pojo;

import lombok.Data;
import lombok.NoArgsConstructor;


@Data
@NoArgsConstructor
public class HotelDoc {
    private Long id;
    private String name;
    private String address;
    private Integer price;
    private Integer score;
    private String brand;
    private String city;
    private String starName;
    private String business;
    private String location;
    private String pic;
    // 排序时的 距离值
    private Object distance;

    public HotelDoc(Hotel hotel) {
        this.id = hotel.getId();
        this.name = hotel.getName();
        this.address = hotel.getAddress();
        this.price = hotel.getPrice();
        this.score = hotel.getScore();
        this.brand = hotel.getBrand();
        this.city = hotel.getCity();
        this.starName = hotel.getStarName();
        this.business = hotel.getBusiness();
        this.location = hotel.getLatitude() + ", " + hotel.getLongitude();
        this.pic = hotel.getPic();
    }
}

```



2）修改HotelService中的handleResponse方法

![image-20210722100613966](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722100613966.png)



重启后测试，发现页面能成功显示距离了：

![image-20210722100838604](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722100838604.png)





### 酒店竞价排名

需求：让指定的酒店在搜索结果中排名置顶

#### 需求分析

要让指定酒店在搜索结果中排名置顶，效果如图：

![image-20210722100947292](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722100947292.png)

页面会给指定的酒店添加**广告**标记。



那怎样才能让指定的酒店排名置顶呢？



我们之前学习过的function_score查询可以影响算分，算分高了，自然排名也就高了。而function_score包含3个要素：

- 过滤条件：哪些文档要加分
- 算分函数：如何计算function score
- 加权方式：function score 与 query score如何运算



这里的需求是：让**指定酒店**排名靠前。因此我们需要给这些酒店添加一个标记，这样在过滤条件中就可以**根据这个标记来判断，是否要提高算分**。

比如，我们给酒店添加一个字段：isAD，Boolean类型：

- true：是广告
- false：不是广告

这样function_score包含3个要素就很好确定了：

- 过滤条件：判断isAD 是否为true
- 算分函数：我们可以用最简单暴力的weight，固定加权值
- 加权方式：可以用默认的相乘，大大提高算分



因此，业务的实现步骤包括：

1. 给HotelDoc类添加isAD字段，Boolean类型

2. 挑选几个你喜欢的酒店，给它的文档数据添加isAD字段，值为true

3. 修改search方法，添加function score功能，给isAD值为true的酒店增加权重



#### 修改HotelDoc实体

给`cn.itcast.hotel.pojo`包下的HotelDoc类添加isAD字段：

```java
private Boolean isAD; // 广告标记
```



#### 添加广告标记

接下来，我们挑几个酒店，添加isAD字段，设置为true：

```json
POST /hotel/_update/1902197537
{
    "doc": {
        "isAD": true
    }
}
POST /hotel/_update/2056126831
{
    "doc": {
        "isAD": true
    }
}
POST /hotel/_update/1989806195
{
    "doc": {
        "isAD": true
    }
}
POST /hotel/_update/2056105938
{
    "doc": {
        "isAD": true
    }
}
```



#### 添加算分函数查询

接下来我们就要修改查询条件了。之前是用的boolean 查询，现在要改成function_socre查询。



function_score查询结构如下：

![image-20210721191544750](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210721191544750.png)



对应的JavaAPI如下：

![image-20210722102850818](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210722102850818.png)



我们可以将之前写的boolean查询作为**原始查询**条件放到query中，接下来就是添加**过滤条件**、**算分函数**、**加权模式**了。所以原来的代码依然可以沿用。



修改`cn.itcast.hotel.service.impl`包下的`HotelService`类中的`buildBasicQuery`方法，添加算分函数查询：

```java
private void buildBasicQuery(RequestParams params, SearchRequest request) {
    // 1.构建BooleanQuery
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    // 关键字搜索
    String key = params.getKey();
    if (key == null || "".equals(key)) {
        boolQuery.must(QueryBuilders.matchAllQuery());
    } else {
        boolQuery.must(QueryBuilders.matchQuery("all", key));
    }
    // 城市条件
    if (params.getCity() != null && !params.getCity().equals("")) {
        boolQuery.filter(QueryBuilders.termQuery("city", params.getCity()));
    }
    // 品牌条件
    if (params.getBrand() != null && !params.getBrand().equals("")) {
        boolQuery.filter(QueryBuilders.termQuery("brand", params.getBrand()));
    }
    // 星级条件
    if (params.getStarName() != null && !params.getStarName().equals("")) {
        boolQuery.filter(QueryBuilders.termQuery("starName", params.getStarName()));
    }
    // 价格
    if (params.getMinPrice() != null && params.getMaxPrice() != null) {
        boolQuery.filter(QueryBuilders
                         .rangeQuery("price")
                         .gte(params.getMinPrice())
                         .lte(params.getMaxPrice())
                        );
    }

    // 2.算分控制
    FunctionScoreQueryBuilder functionScoreQuery =
        QueryBuilders.functionScoreQuery(
        // 原始查询，相关性算分的查询
        boolQuery,
        // function score的数组
        new FunctionScoreQueryBuilder.FilterFunctionBuilder[]{
            // 其中的一个function score 元素
            new FunctionScoreQueryBuilder.FilterFunctionBuilder(
                // 过滤条件
                QueryBuilders.termQuery("isAD", true),
                // 算分函数
                ScoreFunctionBuilders.weightFactorFunction(10)
            )
        });
    request.source().query(functionScoreQuery);
}
```







## 聚合aggregations

**[聚合（](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)[aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)[）](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)**可以让我们极其方便的实现对数据的统计、分析、运算。例如：

- 什么品牌的手机最受欢迎？
- 这些手机的平均价格、最高价格、最低价格？
- 这些手机每月的销售情况如何？

实现这些统计功能的比数据库的sql要方便的多，而且查询速度非常快，可以实现实时搜索效果。

### 基本概念

Elasticsearch中的聚合，包含多种类型，最常用的两种，一个叫`桶`，一个叫`度量`：

> 桶（bucket）

桶的作用，是按照某种方式对数据进行分组，每一组数据在ES中称为一个`桶`，例如我们根据国籍对人划分，可以得到`中国桶`、`英国桶`，`日本桶`……或者我们按照年龄段对人进行划分：0~10,10~20,20~30,30~40等。

Elasticsearch中提供的划分桶的方式有很多：

- Date Histogram Aggregation：根据日期阶梯分组，例如给定阶梯为周，会自动每周分为一组
- Histogram Aggregation：根据数值阶梯分组，与日期类似
- Terms Aggregation：根据词条内容分组，词条内容完全匹配的为一组
- Range Aggregation：数值和日期的范围分组，指定开始和结束，然后按段分组
- ……

综上所述，我们发现bucket aggregations 只负责对数据进行分组，并不进行计算，因此往往bucket中往往会嵌套另一种聚合：metrics aggregations即度量

> 度量（metrics）

分组完成以后，我们一般会对组中的数据进行聚合运算，例如求平均值、最大、最小、求和等，这些在ES中称为`度量`

比较常用的一些度量聚合方式：

- Avg Aggregation：求平均值
- Max Aggregation：求最大值
- Min Aggregation：求最小值
- Percentiles Aggregation：求百分比
- Stats Aggregation：同时返回avg、max、min、sum、count等
- Sum Aggregation：求和
- Top hits Aggregation：求前几
- Value Count Aggregation：求总数
- ……

为了测试聚合，我们先批量导入一些数据

创建索引：

```json
PUT /cars
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "color": {
        "type": "keyword"
      },
      "make": {
        "type": "keyword"
      }
    }
  }
}
```

`注意`：在ES中，需要进行聚合、排序、过滤的字段其处理方式比较特殊，因此不能被分词。这里我们将color和make这两个文字类型的字段设置为keyword类型，这个类型不会被分词，将来就可以参与聚合

批量导入数据，_bulk代表批量

```json
POST /cars/_bulk
{ "index": {}}
{ "price" : 10000, "color" : "red", "make" : "honda", "sold" : "2014-10-28" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 30000, "color" : "green", "make" : "ford", "sold" : "2014-05-18" }
{ "index": {}}
{ "price" : 15000, "color" : "blue", "make" : "toyota", "sold" : "2014-07-02" }
{ "index": {}}
{ "price" : 12000, "color" : "green", "make" : "toyota", "sold" : "2014-08-19" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 80000, "color" : "red", "make" : "bmw", "sold" : "2014-01-01" }
{ "index": {}}
{ "price" : 25000, "color" : "blue", "make" : "ford", "sold" : "2014-02-12" }
```

结果：（注：这里的"_id"因为未指定，是es自动随机生成的。）

```json
{
  "took" : 1474,
  "errors" : false,
  "items" : [
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "gSIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "giIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "gyIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 2,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "hCIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 3,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "hSIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 4,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "hiIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 5,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "hyIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 6,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "cars",
        "_type" : "_doc",
        "_id" : "iCIT0oABY2btytp3yIaR",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 1,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 7,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}
```

### 聚合为桶

首先，我们按照汽车的颜色`color`来划分`桶`：

```json
GET /cars/_search
{
    "size" : 0,
    "aggs" : { 
        "popular_colors" : { 
            "terms" : { 
              "field" : "color"
            }
        }
    }
}
```

- size： 查询条数，这里设置为0，因为我们不关心搜索到的数据，只关心聚合结果，提高效率
- aggs：声明这是一个聚合查询，是aggregations的缩写
  - popular_colors：给这次聚合起一个名字，任意。
    - terms：划分桶的方式，这里是根据词条划分
      - field：划分桶的字段

结果：

```json
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "popular_colors" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "red",
          "doc_count" : 4
        },
        {
          "key" : "blue",
          "doc_count" : 2
        },
        {
          "key" : "green",
          "doc_count" : 2
        }
      ]
    }
  }
}
```

- hits：查询结果为空，因为我们设置了size为0
- aggregations：聚合的结果
- popular_colors：我们定义的聚合名称
- buckets：查找到的桶，每个不同的color字段值都会形成一个桶
  - key：这个桶对应的color字段的值
  - doc_count：这个桶中的文档数量

通过聚合的结果我们发现，目前红色的小车比较畅销！

### 桶内度量

前面的例子告诉我们每个桶里面的文档数量，这很有用。 但通常，我们的应用需要提供更复杂的文档度量。 例如，每种颜色汽车的平均价格是多少？

因此，我们需要告诉Elasticsearch`使用哪个字段`，`使用何种度量方式`进行运算，这些信息要嵌套在`桶`内，`度量`的运算会基于`桶`内的文档进行

现在，我们为刚刚的聚合结果添加求价格平均值的度量：

```json
GET /cars/_search
{
    "size" : 0,
    "aggs" : { 
        "popular_colors" : { 
            "terms" : { 
              "field" : "color"
            },
            "aggs":{
                "avg_price": { 
                   "avg": {
                      "field": "price" 
                   }
                }
            }
        }
    }
}
```

- aggs：我们在上一个aggs(popular_colors)中添加新的aggs。可见`度量`也是一个聚合，度量是在桶内的聚合
- avg_price：聚合的名称
- avg：度量的类型，这里是求平均值
- field：度量运算的字段

结果：

```json
{
  "took" : 14,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "popular_colors" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "red",
          "doc_count" : 4,
          "avg_price" : {
            "value" : 32500.0
          }
        },
        {
          "key" : "blue",
          "doc_count" : 2,
          "avg_price" : {
            "value" : 20000.0
          }
        },
        {
          "key" : "green",
          "doc_count" : 2,
          "avg_price" : {
            "value" : 21000.0
          }
        }
      ]
    }
  }
}
```

可以看到每个桶中都有自己的`avg_price`字段，这是度量聚合的结果

### 桶内嵌套桶

刚刚的案例中，我们在桶内嵌套度量运算。事实上桶不仅可以嵌套运算， 还可以再嵌套其它桶。也就是说在每个分组中，再分更多组。

比如：我们想统计每种颜色的汽车中，分别属于哪个制造商，按照`make`字段再进行分桶

```json
GET /cars/_search
{
    "size" : 0,
    "aggs" : { 
        "popular_colors" : { 
            "terms" : { 
              "field" : "color"
            },
            "aggs":{
                "avg_price": { 
                   "avg": {
                      "field": "price" 
                   }
                },
                "maker":{
                    "terms":{
                        "field":"make"
                    }
                }
            }
        }
    }
}
```

- 原来的color桶和avg计算我们不变
- maker：在嵌套的aggs下新添一个桶，叫做maker
- terms：桶的划分类型依然是词条
- filed：这里根据make字段进行划分

> 注意：前面的概念说了，Elasticsearch中的聚合，包含多种类型，最常用的两种，一个叫`桶`，一个叫`度量`，这说明什么，`说明了桶是一种聚合，度量是另一种聚合。`因此不能将桶查询和度量写在同一个聚合名下，一种桶查询对应一个聚合名，一种度量也要对应一个聚合名。像上面的这种例子，桶内桶，桶内度量，写查询的时候要注意嵌套关系。

结果

```json
{
  "took" : 9,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "popular_colors" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "red",
          "doc_count" : 4,
          "maker" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "honda",
                "doc_count" : 3
              },
              {
                "key" : "bmw",
                "doc_count" : 1
              }
            ]
          },
          "avg_price" : {
            "value" : 32500.0
          }
        },
        {
          "key" : "blue",
          "doc_count" : 2,
          "maker" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "ford",
                "doc_count" : 1
              },
              {
                "key" : "toyota",
                "doc_count" : 1
              }
            ]
          },
          "avg_price" : {
            "value" : 20000.0
          }
        },
        {
          "key" : "green",
          "doc_count" : 2,
          "maker" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "ford",
                "doc_count" : 1
              },
              {
                "key" : "toyota",
                "doc_count" : 1
              }
            ]
          },
          "avg_price" : {
            "value" : 21000.0
          }
        }
      ]
    }
  }
}
```

- 我们可以看到，新的聚合`maker`被嵌套在原来每一个`color`的桶中。
- 每个颜色下面都根据 `make`字段进行了分组
- 我们能读取到的信息：
  - 红色车共有4辆
  - 红色车的平均售价是 $32，500 美元。
  - 其中3辆是 Honda 本田制造，1辆是 BMW 宝马制造。

### 划分桶的其它方式

前面讲了，划分桶的方式有很多，例如：

- Date Histogram Aggregation：根据日期阶梯分组，例如给定阶梯为周，会自动每周分为一组
- Histogram Aggregation：根据数值阶梯分组，与日期类似
- Terms Aggregation：根据词条内容分组，词条内容完全匹配的为一组
- Range Aggregation：数值和日期的范围分组，指定开始和结束，然后按段分组

刚刚的案例中，我们采用的是Terms Aggregation，即根据词条划分桶。

接下来，我们再学习几个比较实用的：

#### 阶梯分桶Histogram

> 原理：

histogram是把数值类型的字段，按照一定的阶梯大小进行分组。你需要指定一个阶梯值（interval）来划分阶梯大小。

举例：

比如你有价格字段，如果你设定interval的值为200，那么阶梯就会是这样的：

0，200，400，600，...

上面列出的是每个阶梯的key，也是区间的启点。

如果一件商品的价格是450，会落入哪个阶梯区间呢？计算公式如下：

```fallback
bucket_key = Math.floor((value - offset) / interval) * interval + offset
```

value：就是当前数据的值，本例中是450

offset：起始偏移量，默认为0

interval：阶梯间隔，比如200

因此你得到的key = Math.floor((450 - 0) / 200) * 200 + 0 = 400

> 操作一下：

比如，我们对汽车的价格进行分组，指定间隔interval为5000：

```json
GET /cars/_search
{
  "size":0,
  "aggs":{
    "price":{
      "histogram": {
        "field": "price",
        "interval": 5000
      }
    }
  }
}
```

结果

```json
{
  "took" : 24,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "price" : {
      "buckets" : [
        {
          "key" : 10000.0,
          "doc_count" : 2
        },
        {
          "key" : 15000.0,
          "doc_count" : 1
        },
        {
          "key" : 20000.0,
          "doc_count" : 2
        },
        {
          "key" : 25000.0,
          "doc_count" : 1
        },
        {
          "key" : 30000.0,
          "doc_count" : 1
        },
        {
          "key" : 35000.0,
          "doc_count" : 0
        },
        {
          "key" : 40000.0,
          "doc_count" : 0
        },
        {
          "key" : 45000.0,
          "doc_count" : 0
        },
        {
          "key" : 50000.0,
          "doc_count" : 0
        },
        {
          "key" : 55000.0,
          "doc_count" : 0
        },
        {
          "key" : 60000.0,
          "doc_count" : 0
        },
        {
          "key" : 65000.0,
          "doc_count" : 0
        },
        {
          "key" : 70000.0,
          "doc_count" : 0
        },
        {
          "key" : 75000.0,
          "doc_count" : 0
        },
        {
          "key" : 80000.0,
          "doc_count" : 1
        }
      ]
    }
  }
}
```

你会发现，中间有大量的文档数量为0 的桶，看起来很丑。

我们可以增加一个参数min_doc_count为1，来约束最少文档数量为1，这样文档数量为0的桶会被过滤

示例：

```json
GET /cars/_search
{
  "size":0,
  "aggs":{
    "price":{
      "histogram": {
        "field": "price",
        "interval": 5000,
        "min_doc_count": 1
      }
    }
  }
}
```

结果

```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "price" : {
      "buckets" : [
        {
          "key" : 10000.0,
          "doc_count" : 2
        },
        {
          "key" : 15000.0,
          "doc_count" : 1
        },
        {
          "key" : 20000.0,
          "doc_count" : 2
        },
        {
          "key" : 25000.0,
          "doc_count" : 1
        },
        {
          "key" : 30000.0,
          "doc_count" : 1
        },
        {
          "key" : 80000.0,
          "doc_count" : 1
        }
      ]
    }
  }
}
```

完美！

如果你用kibana将结果变为柱形图，会更好看：

![25](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/25.png)

#### 范围分桶range

范围分桶与阶梯分桶类似，也是把数字按照阶段进行分组，只不过range方式需要你自己指定每一组的起始和结束大小。

### 总结

聚合常见的有三类：

- **桶（Bucket）**聚合：用来对文档做分组
  - TermAggregation：按照文档字段值分组，例如按照品牌值分组、按照国家分组
  - Date Histogram：按照日期阶梯分组，例如一周为一组，或者一月为一组

- **度量（Metric）**聚合：用以计算一些值，比如：最大值、最小值、平均值等
  - Avg：求平均值
  - Max：求最大值
  - Min：求最小值
  - Stats：同时求max、min、avg、sum等
- **管道（pipeline）**聚合：其它聚合的结果为基础做聚合

> **注意：**参加聚合的字段必须是keyword、日期、数值、布尔类型



## 黑马旅游案例

黑马旅游案例中的聚合需求。

### DSL实现聚合

现在，我们要统计所有数据中的酒店品牌有几种，其实就是按照品牌对数据分组。此时可以根据酒店品牌的名称做聚合，也就是Bucket聚合。

#### Bucket聚合语法

语法如下：

```json
GET /hotel/_search
{
  "size": 0,  // 设置size为0，结果中不包含文档，只包含聚合结果
  "aggs": { // 定义聚合
    "brandAgg": { //给聚合起个名字
      "terms": { // 聚合的类型，按照品牌值聚合，所以选择term
        "field": "brand", // 参与聚合的字段
        "size": 20 // 希望获取的聚合结果数量
      }
    }
  }
}
```

结果如图：

![image-20210723171948228](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723171948228.png)



#### 聚合结果排序

默认情况下，Bucket聚合会统计Bucket内的文档数量，记为_count，并且按照_count降序排序。

我们可以指定order属性，自定义聚合的排序方式：

```json
GET /hotel/_search
{
  "size": 0, 
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "order": {
          "_count": "asc" // 按照_count升序排列
        },
        "size": 20
      }
    }
  }
}
```



#### 限定聚合范围

默认情况下，Bucket聚合是对索引库的所有文档做聚合，但真实场景下，用户会输入搜索条件，因此聚合必须是对搜索结果聚合。那么聚合必须添加限定条件。

我们可以限定要聚合的文档范围，只要添加query条件即可：

```json
GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        "lte": 200 // 只对200元以下的文档聚合
      }
    }
  }, 
  "size": 0, 
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 20
      }
    }
  }
}
```



这次，聚合得到的品牌明显变少了：

![image-20210723172404836](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723172404836.png)



#### Metric聚合语法

前面我们对酒店按照品牌分组，形成了一个个桶。现在我们需要对桶内的酒店做运算，获取每个品牌的用户评分的min、max、avg等值。

这就要用到Metric聚合了，例如stat聚合：就可以获取min、max、avg等结果。

语法如下：

```json
GET /hotel/_search
{
  "size": 0, 
  "aggs": {
    "brandAgg": { 
      "terms": { 
        "field": "brand", 
        "size": 20
      },
      "aggs": { // 是brands聚合的子聚合，也就是分组后对每组分别计算
        "score_stats": { // 聚合名称
          "stats": { // 聚合类型，这里stats可以计算min、max、avg等
            "field": "score" // 聚合字段，这里是score
          }
        }
      }
    }
  }
}
```



这次的score_stats聚合是在brandAgg的聚合内部嵌套的子聚合。因为我们需要在每个桶分别计算。



另外，我们还可以给聚合结果做个排序，例如按照每个桶的酒店平均分做排序：

![image-20210723172917636](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723172917636.png)





#### 小结

aggs代表聚合，与query同级，此时query的作用是？

- 限定聚合的的文档范围

聚合必须的三要素：

- 聚合名称
- 聚合类型
- 聚合字段

聚合可配置属性有：

- size：指定聚合结果数量
- order：指定聚合结果排序方式
- field：指定聚合字段



### RestAPI实现聚合

#### API语法

聚合条件与query条件同级别，因此需要使用request.source()来指定聚合条件。

聚合条件的语法：

![image-20210723173057733](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723173057733.png)



聚合的结果也与查询结果不同，API也比较特殊。不过同样是JSON逐层解析：

![image-20210723173215728](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723173215728.png)



#### 业务需求

需求：搜索页面的品牌、城市等信息不应该是在页面写死，而是通过聚合索引库中的酒店数据得来的：

![image-20210723192605566](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723192605566.png)



分析：

目前，页面的城市列表、星级列表、品牌列表都是写死的，并不会随着搜索结果的变化而变化。但是用户搜索条件改变时，搜索结果会跟着变化。

例如：用户搜索“东方明珠”，那搜索的酒店肯定是在上海东方明珠附近，因此，城市只能是上海，此时城市列表中就不应该显示北京、深圳、杭州这些信息了。



也就是说，搜索结果中包含哪些城市，页面就应该列出哪些城市；搜索结果中包含哪些品牌，页面就应该列出哪些品牌。

如何得知搜索结果中包含哪些品牌？如何得知搜索结果中包含哪些城市？



使用聚合功能，利用Bucket聚合，对搜索结果中的文档基于品牌分组、基于城市分组，就能得知包含哪些品牌、哪些城市了。

因为是对搜索结果聚合，因此聚合是**限定范围的聚合**，也就是说聚合的限定条件跟搜索文档的条件一致。



查看浏览器可以发现，前端其实已经发出了这样的一个请求：

![image-20210723193730799](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723193730799.png)

请求**参数与搜索文档的参数完全一致**。



返回值类型就是页面要展示的最终结果：

![image-20210723203915982](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723203915982.png)

结果是一个Map结构：

- key是字符串，城市、星级、品牌、价格
- value是集合，例如多个城市的名称



#### 业务实现

在`cn.itcast.hotel.web`包的`HotelController`中添加一个方法，遵循下面的要求：

- 请求方式：`POST`
- 请求路径：`/hotel/filters`
- 请求参数：`RequestParams`，与搜索文档的参数一致
- 返回值类型：`Map<String, List<String>>`

代码：

```java
    @PostMapping("filters")
    public Map<String, List<String>> getFilters(@RequestBody RequestParams params){
        return hotelService.getFilters(params);
    }
```



这里调用了IHotelService中的getFilters方法，尚未实现。

在`cn.itcast.hotel.service.IHotelService`中定义新方法：

```java
Map<String, List<String>> filters(RequestParams params);
```



在`cn.itcast.hotel.service.impl.HotelService`中实现该方法：

```java
@Override
public Map<String, List<String>> filters(RequestParams params) {
    try {
        // 1.准备Request
        SearchRequest request = new SearchRequest("hotel");
        // 2.准备DSL
        // 2.1.query
        buildBasicQuery(params, request);
        // 2.2.设置size
        request.source().size(0);
        // 2.3.聚合
        buildAggregation(request);
        // 3.发出请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 4.解析结果
        Map<String, List<String>> result = new HashMap<>();
        Aggregations aggregations = response.getAggregations();
        // 4.1.根据品牌名称，获取品牌结果
        List<String> brandList = getAggByName(aggregations, "brandAgg");
        result.put("品牌", brandList);
        // 4.2.根据品牌名称，获取品牌结果
        List<String> cityList = getAggByName(aggregations, "cityAgg");
        result.put("城市", cityList);
        // 4.3.根据品牌名称，获取品牌结果
        List<String> starList = getAggByName(aggregations, "starAgg");
        result.put("星级", starList);

        return result;
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}

private void buildAggregation(SearchRequest request) {
    request.source().aggregation(AggregationBuilders
                                 .terms("brandAgg")
                                 .field("brand")
                                 .size(100)
                                );
    request.source().aggregation(AggregationBuilders
                                 .terms("cityAgg")
                                 .field("city")
                                 .size(100)
                                );
    request.source().aggregation(AggregationBuilders
                                 .terms("starAgg")
                                 .field("starName")
                                 .size(100)
                                );
}

private List<String> getAggByName(Aggregations aggregations, String aggName) {
    // 4.1.根据聚合名称获取聚合结果
    Terms brandTerms = aggregations.get(aggName);
    // 4.2.获取buckets
    List<? extends Terms.Bucket> buckets = brandTerms.getBuckets();
    // 4.3.遍历
    List<String> brandList = new ArrayList<>();
    for (Terms.Bucket bucket : buckets) {
        // 4.4.获取key
        String key = bucket.getKeyAsString();
        brandList.add(key);
    }
    return brandList;
}
```





## 自动补全

当用户在搜索框输入字符时，我们应该提示出与该字符有关的搜索项，如图：

![image-20210723204936367](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723204936367.png)

这种根据用户输入的字母，提示完整词条的功能，就是自动补全了。



因为需要根据拼音字母来推断，因此要用到拼音分词功能。



### 拼音分词器



要实现根据字母做补全，就必须对文档按照拼音分词。在GitHub上恰好有elasticsearch的拼音分词插件。地址：https://github.com/medcl/elasticsearch-analysis-pinyin

![image-20210723205932746](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723205932746.png)

拼音分词器的安装包的安装方式与IK分词器一样，分三步：

​	①解压

​	②上传到虚拟机中，elasticsearch的plugin目录

​	③重启elasticsearch

​	④测试

详细安装步骤可以参考IK分词器的安装过程。





测试用法如下：

```json
POST /_analyze
{
  "text": "如家酒店还不错",
  "analyzer": "pinyin"
}
```

结果：

![image-20210723210126506](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723210126506.png) 





### 自定义分词器

默认的拼音分词器会将每个汉字单独分为拼音，而我们希望的是每个词条形成一组拼音，需要对拼音分词器做个性化定制，形成自定义分词器。



elasticsearch中分词器（analyzer）的组成包含三部分：

- character filters：在tokenizer之前对文本进行处理。例如删除字符、替换字符
- tokenizer：将文本按照一定的规则切割成词条（term）。例如keyword，就是不分词；还有ik_smart
- tokenizer filter：将tokenizer输出的词条做进一步处理。例如大小写转换、同义词处理、拼音处理等



文档分词时会依次由这三部分来处理文档：

   ![image-20210723210427878](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723210427878.png)

声明自定义分词器的语法如下：

```json
PUT /test
{
  "settings": {
    "analysis": {
      "analyzer": { // 自定义分词器
        "my_analyzer": {  // 分词器名称
          "tokenizer": "ik_max_word",
          "filter": "py"
        }
      },
      "filter": { // 自定义tokenizer filter
        "py": { // 过滤器名称
          "type": "pinyin", // 过滤器类型，这里是pinyin
		  "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_analyzer",
        "search_analyzer": "ik_smart"
      }
    }
  }
}
```

测试：

```json
POST /test/_analyze
{
  "text": "如家酒店还不错",
  "analyzer": "my_analyzer"
}
```

结果：

```json
{
  "tokens" : [
    {
      "token" : "如家",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "rujia",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "rj",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "酒店",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "jiudian",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "jd",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "还不",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "haibu",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "hb",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "不错",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "bucuo",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "bc",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 3
    }
  ]
}

```

如图：

![image-20210723211829150](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723211829150.png)





总结：

如何使用拼音分词器？

- ①下载pinyin分词器

- ②解压并放到elasticsearch的plugin目录

- ③重启即可

如何自定义分词器？

- ①创建索引库时，在settings中配置，可以包含三部分

- ②character filter

- ③tokenizer

- ④filter

拼音分词器注意事项？

- 为了避免搜索到同音字，搜索时不要使用拼音分词器





### 自动补全查询

elasticsearch提供了[Completion Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-suggesters.html)查询来实现自动补全功能。这个查询会匹配以用户输入内容开头的词条并返回。为了提高补全查询的效率，对于文档中字段的类型有一些约束：

- 参与补全查询的字段必须是completion类型。

- 字段的内容一般是用来补全的多个词条形成的数组。

比如，一个这样的索引库：

```json
// 创建索引库
PUT test
{
  "mappings": {
    "properties": {
      "title":{
        "type": "completion"
      }
    }
  }
}
```

然后插入下面的数据：

```json
// 示例数据
POST test/_doc
{
  "title": ["Sony", "WH-1000XM3"]
}
POST test/_doc
{
  "title": ["SK-II", "PITERA"]
}
POST test/_doc
{
  "title": ["Nintendo", "switch"]
}
```



查询的DSL语句如下：

```json
// 自动补全查询
GET /test/_search
{
  "suggest": {
    "title_suggest": {
      "text": "s", // 关键字
      "completion": {
        "field": "title", // 补全查询的字段
        "skip_duplicates": true, // 跳过重复的
        "size": 10 // 获取前10条结果
      }
    }
  }
}
```







### 实现酒店搜索框自动补全

现在，我们的hotel索引库还没有设置拼音分词器，需要修改索引库中的配置。但是我们知道索引库是无法修改的，只能删除然后重新创建。

另外，我们需要添加一个字段，用来做自动补全，将brand、suggestion、city等都放进去，作为自动补全的提示。



因此，总结一下，我们需要做的事情包括：

1. 修改hotel索引库结构，设置自定义拼音分词器

2. 修改索引库的name、all字段，使用自定义分词器

3. 索引库添加一个新字段suggestion，类型为completion类型，使用自定义的分词器

4. 给HotelDoc类添加suggestion字段，内容包含brand、business

5. 重新导入数据到hotel库



#### 修改酒店映射结构

代码如下：

```json
// 酒店数据索引库
PUT /hotel
{
  "settings": {
    "analysis": {
      "analyzer": {
        "text_anlyzer": {
          "tokenizer": "ik_max_word",
          "filter": "py"
        },
        "completion_analyzer": {
          "tokenizer": "keyword",
          "filter": "py"
        }
      },
      "filter": {
        "py": {
          "type": "pinyin",
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id":{
        "type": "keyword"
      },
      "name":{
        "type": "text",
        "analyzer": "text_anlyzer",
        "search_analyzer": "ik_smart",
        "copy_to": "all"
      },
      "address":{
        "type": "keyword",
        "index": false
      },
      "price":{
        "type": "integer"
      },
      "score":{
        "type": "integer"
      },
      "brand":{
        "type": "keyword",
        "copy_to": "all"
      },
      "city":{
        "type": "keyword"
      },
      "starName":{
        "type": "keyword"
      },
      "business":{
        "type": "keyword",
        "copy_to": "all"
      },
      "location":{
        "type": "geo_point"
      },
      "pic":{
        "type": "keyword",
        "index": false
      },
      "all":{
        "type": "text",
        "analyzer": "text_anlyzer",
        "search_analyzer": "ik_smart"
      },
      "suggestion":{
          "type": "completion",
          "analyzer": "completion_analyzer"
      }
    }
  }
}
```



#### 修改HotelDoc实体

HotelDoc中要添加一个字段，用来做自动补全，内容可以是酒店品牌、城市、商圈等信息。按照自动补全字段的要求，最好是这些字段的数组。

因此我们在HotelDoc中添加一个suggestion字段，类型为`List<String>`，然后将brand、city、business等信息放到里面。

代码如下：

```java
package cn.itcast.hotel.pojo;

import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

@Data
@NoArgsConstructor
public class HotelDoc {
    private Long id;
    private String name;
    private String address;
    private Integer price;
    private Integer score;
    private String brand;
    private String city;
    private String starName;
    private String business;
    private String location;
    private String pic;
    private Object distance;
    private Boolean isAD;
    private List<String> suggestion;

    public HotelDoc(Hotel hotel) {
        this.id = hotel.getId();
        this.name = hotel.getName();
        this.address = hotel.getAddress();
        this.price = hotel.getPrice();
        this.score = hotel.getScore();
        this.brand = hotel.getBrand();
        this.city = hotel.getCity();
        this.starName = hotel.getStarName();
        this.business = hotel.getBusiness();
        this.location = hotel.getLatitude() + ", " + hotel.getLongitude();
        this.pic = hotel.getPic();
        // 组装suggestion
        if(this.business.contains("/")){
            // business有多个值，需要切割
            String[] arr = this.business.split("/");
            // 添加元素
            this.suggestion = new ArrayList<>();
            this.suggestion.add(this.brand);
            Collections.addAll(this.suggestion, arr);
        }else {
            this.suggestion = Arrays.asList(this.brand, this.business);
        }
    }
}
```



#### 重新导入

重新执行之前编写的导入数据功能，可以看到新的酒店数据中包含了suggestion：

![image-20210723213546183](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723213546183.png)





#### 自动补全查询的JavaAPI

之前我们学习了自动补全查询的DSL，而没有学习对应的JavaAPI，这里给出一个示例：

![image-20210723213759922](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723213759922.png)



而自动补全的结果也比较特殊，解析的代码如下：

![image-20210723213917524](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723213917524.png)



#### 实现搜索框自动补全

查看前端页面，可以发现当我们在输入框键入时，前端会发起ajax请求：

![image-20210723214021062](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723214021062.png)

返回值是补全词条的集合，类型为`List<String>`



1）在`cn.itcast.hotel.web`包下的`HotelController`中添加新接口，接收新的请求：

```java
@GetMapping("suggestion")
public List<String> getSuggestions(@RequestParam("key") String prefix) {
    return hotelService.getSuggestions(prefix);
}
```



2）在`cn.itcast.hotel.service`包下的`IhotelService`中添加方法：

```java
List<String> getSuggestions(String prefix);
```



3）在`cn.itcast.hotel.service.impl.HotelService`中实现该方法：

```java
@Override
public List<String> getSuggestions(String prefix) {
    try {
        // 1.准备Request
        SearchRequest request = new SearchRequest("hotel");
        // 2.准备DSL
        request.source().suggest(new SuggestBuilder().addSuggestion(
            "suggestions",
            SuggestBuilders.completionSuggestion("suggestion")
            .prefix(prefix)
            .skipDuplicates(true)
            .size(10)
        ));
        // 3.发起请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 4.解析结果
        Suggest suggest = response.getSuggest();
        // 4.1.根据补全查询名称，获取补全结果
        CompletionSuggestion suggestions = suggest.getSuggestion("suggestions");
        // 4.2.获取options
        List<CompletionSuggestion.Entry.Option> options = suggestions.getOptions();
        // 4.3.遍历
        List<String> list = new ArrayList<>(options.size());
        for (CompletionSuggestion.Entry.Option option : options) {
            String text = option.getText().toString();
            list.add(text);
        }
        return list;
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```





## 数据同步

elasticsearch中的酒店数据来自于mysql数据库，因此mysql数据发生改变时，elasticsearch也必须跟着改变，这个就是elasticsearch与mysql之间的**数据同步**。



![image-20210723214758392](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723214758392.png)





### 思路分析

常见的数据同步方案有三种：

- 同步调用
- 异步通知
- 监听binlog



#### 同步调用

方案一：同步调用

![image-20210723214931869](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723214931869.png)

基本步骤如下：

- hotel-demo对外提供接口，用来修改elasticsearch中的数据
- 酒店管理服务在完成数据库操作后，直接调用hotel-demo提供的接口，



#### 异步通知

方案二：异步通知

![image-20210723215140735](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723215140735.png)



流程如下：

- hotel-admin对mysql数据库数据完成增、删、改后，发送MQ消息
- hotel-demo监听MQ，接收到消息后完成elasticsearch数据修改





#### 监听binlog

方案三：监听binlog

![image-20210723215518541](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723215518541.png)

流程如下：

- 给mysql开启binlog功能
- mysql完成增、删、改操作都会记录在binlog中
- hotel-demo基于canal监听binlog变化，实时更新elasticsearch中的内容



#### 选择

方式一：同步调用

- 优点：实现简单，粗暴
- 缺点：业务耦合度高

方式二：异步通知

- 优点：低耦合，实现难度一般
- 缺点：依赖mq的可靠性

方式三：监听binlog

- 优点：完全解除服务间耦合
- 缺点：开启binlog增加数据库负担、实现复杂度高



### 实现数据同步



#### 思路

hotel-admin项目作为酒店管理的微服务。当酒店数据发生增、删、改时，要求对elasticsearch中数据也要完成相同操作。

步骤：

- hotel-admin项目，启动并测试酒店数据的CRUD

- 声明exchange、queue、RoutingKey

- 在hotel-admin中的增、删、改业务中完成消息发送

- 在hotel-demo中完成消息监听，并更新elasticsearch中数据

- 启动并测试数据同步功能

hotel-admin项目其中包含了酒店的CRUD功能。

#### 声明交换机、队列

MQ结构如图：

![image-20210723215850307](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723215850307.png)



##### 1）引入依赖

在hotel-admin、hotel-demo中引入rabbitmq的依赖：

```xml
<!--amqp-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

并且修改这两个微服务的配置文件，添加：

```yml
spring:
  rabbitmq:
    host: localhost
    username: root
    password: 123456
    virtual-host: hotel
```



##### 2）声明队列交换机名称

在hotel-admin和hotel-demo中的`cn.itcast.hotel.constatnts`包下新建一个类`MqConstants`：

```java
package cn.itcast.hotel.constatnts;

    public class MqConstants {
    /**
     * 交换机
     */
    public final static String HOTEL_EXCHANGE = "hotel.topic";
    /**
     * 监听新增和修改的队列
     */
    public final static String HOTEL_INSERT_QUEUE = "hotel.insert.queue";
    /**
     * 监听删除的队列
     */
    public final static String HOTEL_DELETE_QUEUE = "hotel.delete.queue";
    /**
     * 新增或修改的RoutingKey
     */
    public final static String HOTEL_INSERT_KEY = "hotel.insert";
    /**
     * 删除的RoutingKey
     */
    public final static String HOTEL_DELETE_KEY = "hotel.delete";
}
```



##### 3）声明队列交换机

在hotel-demo中，定义配置类，声明队列、交换机：

```java
package cn.itcast.hotel.config;

import cn.itcast.hotel.constants.MqConstants;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqConfig {
    @Bean
    public TopicExchange topicExchange(){
        return new TopicExchange(MqConstants.HOTEL_EXCHANGE, true, false);
    }

    @Bean
    public Queue insertQueue(){
        return new Queue(MqConstants.HOTEL_INSERT_QUEUE, true);
    }

    @Bean
    public Queue deleteQueue(){
        return new Queue(MqConstants.HOTEL_DELETE_QUEUE, true);
    }

    @Bean
    public Binding insertQueueBinding(){
        return BindingBuilder.bind(insertQueue()).to(topicExchange()).with(MqConstants.HOTEL_INSERT_KEY);
    }

    @Bean
    public Binding deleteQueueBinding(){
        return BindingBuilder.bind(deleteQueue()).to(topicExchange()).with(MqConstants.HOTEL_DELETE_KEY);
    }
}
```



#### 发送MQ消息

在hotel-admin中的增、删、改业务中分别发送MQ消息：

![image-20210723221843816](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723221843816.png)



#### 接收MQ消息

hotel-demo接收到MQ消息要做的事情包括：

- 新增消息：根据传递的hotel的id查询hotel信息，然后新增一条数据到索引库
- 删除消息：根据传递的hotel的id删除索引库中的一条数据



1）首先在hotel-demo的`cn.itcast.hotel.service`包下的`IHotelService`中新增新增、删除业务

```java
void deleteById(Long id);

void insertById(Long id);
```



2）给hotel-demo中的`cn.itcast.hotel.service.impl`包下的HotelService中实现业务：

```java
@Override
public void deleteById(Long id) {
    try {
        // 1.准备Request
        DeleteRequest request = new DeleteRequest("hotel", id.toString());
        // 2.发送请求
        client.delete(request, RequestOptions.DEFAULT);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}

@Override
public void insertById(Long id) {
    try {
        // 0.根据id查询酒店数据
        Hotel hotel = getById(id);
        // 转换为文档类型
        HotelDoc hotelDoc = new HotelDoc(hotel);

        // 1.准备Request对象
        IndexRequest request = new IndexRequest("hotel").id(hotel.getId().toString());
        // 2.准备Json文档
        request.source(JSON.toJSONString(hotelDoc), XContentType.JSON);
        // 3.发送请求
        client.index(request, RequestOptions.DEFAULT);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```



3）编写监听器

在hotel-demo中的`cn.itcast.hotel.mq`包新增一个类：

```java
package cn.itcast.hotel.mq;

import cn.itcast.hotel.constants.MqConstants;
import cn.itcast.hotel.service.IHotelService;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class HotelListener {

    @Autowired
    private IHotelService hotelService;

    /**
     * 监听酒店新增或修改的业务
     * @param id 酒店id
     */
    @RabbitListener(queues = MqConstants.HOTEL_INSERT_QUEUE)
    public void listenHotelInsertOrUpdate(Long id){
        hotelService.insertById(id);
    }

    /**
     * 监听酒店删除的业务
     * @param id 酒店id
     */
    @RabbitListener(queues = MqConstants.HOTEL_DELETE_QUEUE)
    public void listenHotelDelete(Long id){
        hotelService.deleteById(id);
    }
}
```









## Spring Data Elasticsearch

Elasticsearch提供的Java客户端有一些不太方便的地方：

- 很多地方需要拼接Json字符串，在java中拼接字符串有多恐怖你应该懂的
- 需要自己把对象序列化为json存储
- 查询到结果也需要自己反序列化为对象

因此，就有了套件：Spring Data Elasticsearch。

**spring-data-Elasticsearch 使用之前，必须先确定版本，elasticsearch 对版本的要求特别高！**

### 简介

Spring Data Elasticsearch是Spring Data项目下的一个子模块。

查看 Spring Data的官网：http://projects.spring.io/spring-data/

![26](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/26.png)

> Spring Data的使命是为数据访问提供熟悉且一致的基于Spring的编程模型，同时仍保留底层数据存储的特殊特性。
>
> 它使得使用数据访问技术，关系数据库和非关系数据库，map-reduce框架和基于云的数据服务变得容易。这是一个总括项目，其中包含许多特定于给定数据库的子项目。这些令人兴奋的技术项目背后，是由许多公司和开发人员合作开发的。

Spring Data 的使命是给各种数据访问提供统一的编程接口，不管是关系型数据库（如MySQL），还是非关系数据库（如Redis），或者类似Elasticsearch这样的索引数据库。从而简化开发人员的代码，提高开发效率。

包含很多不同数据操作的模块：

![27](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/27.png)

Spring Data Elasticsearch的页面：https://projects.spring.io/spring-data-elasticsearch/

![28](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/28.png)

特征：

- 支持Spring的基于`@Configuration`的java配置方式，或者XML配置方式
- 提供了用于操作ES的便捷工具类**`ElasticsearchTemplate`**。包括实现文档到POJO之间的自动智能映射。
- 利用Spring的数据转换服务实现的功能丰富的对象映射
- 基于注解的元数据映射方式，而且可扩展以支持更多不同的数据格式
- 根据持久层接口自动生成对应实现方法，无需人工编写基本操作代码（类似mybatis，根据接口自动得到实现）。当然，也支持人工定制查询

### 创建Demo工程

我们新建一个demo，学习Elasticsearch

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.chan</groupId>
    <artifactId>es-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <!-- 这是Spring Boot的父级依赖，这样当前的项目就是Spring Boot项目了。spring-boot-starter-parent是一个特殊的starter，
    它用来提供相关的Maven默认依赖。使用它之后，常用的包依赖可以省去version标签 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

```yml
spring:
  data:
    elasticsearch:
      cluster-name: elasticsearch
      cluster-nodes: 192.168.1.104:9300
```

```java
/**
 * @Classname EsApplication
 * @Description TODO
 * @Date 2021/2/25 19:09
 * @Created by ChenSiYu
 */
@SpringBootApplication
public class EsApplication {
    public static void main(String[] args) {
        SpringApplication.run(EsApplication.class, args);
    }
}
```

### 实体类及注解

首先我们准备好实体类：

```java
public class Item {
    Long id;
    String title; //标题
    String category;// 分类
    String brand; // 品牌
    Double price; // 价格
    String images; // 图片地址
}
```

> 映射

Spring Data通过注解来声明字段的映射属性，有下面的三个注解：

- `@Document`
  
  作用在类，标记实体类为文档对象，一般有两个属性

  - indexName：对应索引名称
  - type：对应在索引中的类型（新版本中已舍弃，不用写它。）
  - shards：分片数量，默认5
  - replicas：副本数量，默认1
  
- `@Id` 作用在成员变量，标记一个字段作为id主键

- `@Field` 
  
  作用在成员变量，标记为文档的字段，并指定字段映射属性：

  - type：字段类型，取值是枚举：FieldType
  - index：是否索引，布尔类型，默认是true
  - store：是否存储，布尔类型，默认是false
  - analyzer：分词器名称

示例：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Document(indexName = "item", shards = 1, replicas = 0)
public class Item {
    @Id
    @Field(type = FieldType.Long)
    private Long id;// 如果id不指定，es会随机生成一个文档id

    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String title; //标题

    @Field(type = FieldType.Keyword)
    private String category;// 分类

    @Field(type = FieldType.Keyword)
    private String brand; // 品牌

    @Field(type = FieldType.Double)
    private Double price; // 价格

    @Field(type = FieldType.Keyword, index = false)
    private String images; // 图片地址
}
```

### Template索引操作

创建Spring Boot测试类。

#### 创建索引和映射

> 索引

ElasticsearchTemplate中提供了创建索引的API：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/29.png)

可以根据类的信息自动生成，也可以手动指定indexName和Settings

> 映射

映射相关的API：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/30.png)

可以根据类的字节码信息（注解配置）来生成映射，或者手动编写映射

我们这里采用类的字节码信息创建索引并映射：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = EsApplication.class)// 就是你springboot的启动类
public class EsTest {

    // ElasticsearchTemplate一般用来处理Repository文档操作处理不了的需求
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;

    @Autowired
    private ItemRepository itemRepository;

    /**
     * @Description 测试创建索引并创建映射
     * @Param []
     * @return void
     * @Date 2021/2/25 21:09
     * @Author ChenSiYu
     */
    @Test
    public void testCreate(){
        // 创建索引 会根据Item类的@Document注解信息来创建
        elasticsearchTemplate.createIndex(Item.class);
        // 创建映射 会根据Item类中的id、Field等字段来自动完成映射
        elasticsearchTemplate.putMapping(Item.class);
    }
}
```

GET /item 的结果：

```json
{
  "item" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "brand" : {
          "type" : "keyword"
        },
        "category" : {
          "type" : "keyword"
        },
        "id" : {
          "type" : "keyword"
        },
        "images" : {
          "type" : "keyword",
          "index" : false
        },
        "price" : {
          "type" : "double"
        },
        "title" : {
          "type" : "text",
          "analyzer" : "ik_max_word"
        }
      }
    },
    "settings" : {
      "index" : {
        "refresh_interval" : "1s",
        "number_of_shards" : "1",
        "provided_name" : "item",
        "creation_date" : "1652763123385",
        "store" : {
          "type" : "fs"
        },
        "number_of_replicas" : "0",
        "uuid" : "YZLIiLacRJqEMWAARDhl8A",
        "version" : {
          "created" : "7070099"
        }
      }
    }
  }
}
```

#### 删除索引

删除索引的API：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/31.png)

可以根据类名或索引名删除。

示例：

```java
    @Test
    public void testDelete(){
        // 两种都可以
//        elasticsearchTemplate.deleteIndex("item");
        elasticsearchTemplate.deleteIndex(Item.class);
    }
```

### Repository文档操作

Spring Data 的强大之处，就在于你不用写任何DAO处理，自动根据方法名或类的信息进行CRUD操作。只要你定义一个接口，然后继承Repository提供的一些子接口，就能具备各种基本的CRUD功能。

我们只需要定义接口，然后继承它就OK了。

```java
public interface ItemRepository extends ElasticsearchRepository<Item, Long> {
}
```

来看下Repository的继承关系：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/32.png)

我们看到有一个ElasticsearchRepository接口：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/33.png)

#### 新增文档

```java
@Autowired
private ItemRepository itemRepository;

@Test
public void addDoc(){
    Item item = new Item(1L, "小米手机7", " 手机", "小米", 3499.00, "http://image.leyou.com/13123.jpg");
    // save方法有增又有改 好比PUT
    itemRepository.save(item);
}
```

去页面查询看看：

```json
GET /item/_search
{
  "query": {
    "match_all": {}
  }
}
```

结果：

```json
{
  "took": 14,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1,
    "hits": [
      {
        "_index": "item",
        "_type": "item",
        "_id": "1",
        "_score": 1,
        "_source": {
          "id": 1,
          "title": "小米手机7",
          "category": " 手机",
          "brand": "小米",
          "price": 3499,
          "images": "http://image.leyou.com/13123.jpg"
        }
      }
    ]
  }
}
```

#### 批量新增

```java
@Test
public void addDocs(){
    List<Item> list = new ArrayList<>();
    list.add(new Item(2L, "坚果手机R1", " 手机", "锤子", 3699.00, "http://image.leyou.com/123.jpg"));
    list.add(new Item(3L, "华为META10", " 手机", "华为", 4499.00, "http://image.leyou.com/3.jpg"));
    // 接收对象的集合，实现批量新增
    itemRepository.saveAll(list);
}
```

```json
{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "item",
        "_type": "item",
        "_id": "1",
        "_score": 1,
        "_source": {
          "id": 1,
          "title": "小米手机7",
          "category": " 手机",
          "brand": "小米",
          "price": 3499,
          "images": "http://image.leyou.com/13123.jpg"
        }
      },
      {
        "_index": "item",
        "_type": "item",
        "_id": "2",
        "_score": 1,
        "_source": {
          "id": 2,
          "title": "坚果手机R1",
          "category": " 手机",
          "brand": "锤子",
          "price": 3699,
          "images": "http://image.leyou.com/123.jpg"
        }
      },
      {
        "_index": "item",
        "_type": "item",
        "_id": "3",
        "_score": 1,
        "_source": {
          "id": 3,
          "title": "华为META10",
          "category": " 手机",
          "brand": "华为",
          "price": 4499,
          "images": "http://image.leyou.com/3.jpg"
        }
      }
    ]
  }
}
```

#### 修改文档

修改和新增是同一个接口，区分的依据就是id，这一点跟我们在页面发起PUT请求是类似的。

#### 基本查询

ElasticsearchRepository提供了一些基本的查询方法：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/34.png)

我们来试试查询所有：

```java
@Test
public void testFind(){
    // 查询全部，并按照价格降序排序
    Iterable<Item> items = this.itemRepository.findAll(Sort.by(Sort.Direction.DESC, "price"));
    //        items.forEach(item-> System.out.println(item));
    items.forEach(System.out::println);
}
```

结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/35.png)

#### 自定义方法

Spring Data 的另一个强大功能，是根据方法名称自动实现功能。

比如：你的方法名叫做：findByTitle，那么它就知道你是根据title查询，然后自动帮你完成，无需写实现类。

当然，方法名称要符合一定的约定：

| Keyword               | Sample                                     | Elasticsearch Query String                                   |
| --------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| `And`                 | `findByNameAndPrice`                       | `{"bool" : {"must" : [ {"field" : {"name" : "?"}}, {"field" : {"price" : "?"}} ]}}` |
| `Or`                  | `findByNameOrPrice`                        | `{"bool" : {"should" : [ {"field" : {"name" : "?"}}, {"field" : {"price" : "?"}} ]}}` |
| `Is`                  | `findByName`                               | `{"bool" : {"must" : {"field" : {"name" : "?"}}}}`           |
| `Not`                 | `findByNameNot`                            | `{"bool" : {"must_not" : {"field" : {"name" : "?"}}}}`       |
| `Between`             | `findByPriceBetween`                       | `{"bool" : {"must" : {"range" : {"price" : {"from" : ?,"to" : ?,"include_lower" : true,"include_upper" : true}}}}}` |
| `LessThanEqual`       | `findByPriceLessThan`                      | `{"bool" : {"must" : {"range" : {"price" : {"from" : null,"to" : ?,"include_lower" : true,"include_upper" : true}}}}}` |
| `GreaterThanEqual`    | `findByPriceGreaterThan`                   | `{"bool" : {"must" : {"range" : {"price" : {"from" : ?,"to" : null,"include_lower" : true,"include_upper" : true}}}}}` |
| `Before`              | `findByPriceBefore`                        | `{"bool" : {"must" : {"range" : {"price" : {"from" : null,"to" : ?,"include_lower" : true,"include_upper" : true}}}}}` |
| `After`               | `findByPriceAfter`                         | `{"bool" : {"must" : {"range" : {"price" : {"from" : ?,"to" : null,"include_lower" : true,"include_upper" : true}}}}}` |
| `Like`                | `findByNameLike`                           | `{"bool" : {"must" : {"field" : {"name" : {"query" : "?*","analyze_wildcard" : true}}}}}` |
| `StartingWith`        | `findByNameStartingWith`                   | `{"bool" : {"must" : {"field" : {"name" : {"query" : "?*","analyze_wildcard" : true}}}}}` |
| `EndingWith`          | `findByNameEndingWith`                     | `{"bool" : {"must" : {"field" : {"name" : {"query" : "*?","analyze_wildcard" : true}}}}}` |
| `Contains/Containing` | `findByNameContaining`                     | `{"bool" : {"must" : {"field" : {"name" : {"query" : "**?**","analyze_wildcard" : true}}}}}` |
| `In`                  | `findByNameIn(Collection<String>names)`    | `{"bool" : {"must" : {"bool" : {"should" : [ {"field" : {"name" : "?"}}, {"field" : {"name" : "?"}} ]}}}}` |
| `NotIn`               | `findByNameNotIn(Collection<String>names)` | `{"bool" : {"must_not" : {"bool" : {"should" : {"field" : {"name" : "?"}}}}}}` |
| `Near`                | `findByStoreNear`                          | `Not Supported Yet !`                                        |
| `True`                | `findByAvailableTrue`                      | `{"bool" : {"must" : {"field" : {"available" : true}}}}`     |
| `False`               | `findByAvailableFalse`                     | `{"bool" : {"must" : {"field" : {"available" : false}}}}`    |
| `OrderBy`             | `findByAvailableTrueOrderByNameDesc`       | `{"sort" : [{ "name" : {"order" : "desc"} }],"bool" : {"must" : {"field" : {"available" : true}}}}` |

例如，我们来按照价格区间查询，定义这样的一个方法：

```java
public interface ItemRepository extends ElasticsearchRepository<Item,Long> {

    /**
     * 根据价格区间查询
     * @param price1
     * @param price2
     * @return
     */
    List<Item> findByPriceBetween(double price1, double price2);
}
```

然后再多添加一些测试数据：

```java
@Test
public void indexList() {
    List<Item> list = new ArrayList<>();
    list.add(new Item(1L, "小米手机7", "手机", "小米", 3299.00, "http://image.leyou.com/13123.jpg"));
    list.add(new Item(2L, "坚果手机R1", "手机", "锤子", 3699.00, "http://image.leyou.com/13123.jpg"));
    list.add(new Item(3L, "华为META10", "手机", "华为", 4499.00, "http://image.leyou.com/13123.jpg"));
    list.add(new Item(4L, "小米Mix2S", "手机", "小米", 4299.00, "http://image.leyou.com/13123.jpg"));
    list.add(new Item(5L, "荣耀V10", "手机", "华为", 2799.00, "http://image.leyou.com/13123.jpg"));
    // 接收对象集合，实现批量新增
    itemRepository.saveAll(list);
}
```

不需要写实现类，然后我们直接去运行：

```java
@Test
public void queryByPriceBetween(){
    List<Item> list = this.itemRepository.findByPriceBetween(2000.00, 3500.00);
    for (Item item : list) {
        System.out.println("item = " + item);
    }
}
```

结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/36.png)

虽然基本查询和自定义方法已经很强大了，但是如果是复杂查询（模糊、通配符、词条查询等）就显得力不从心了。所以继续往下看。

### 高级查询

#### 高级查询中的基本查询

先看看基本玩法

```java
@Test
public void testQuery(){
    // 词条查询
    MatchQueryBuilder queryBuilder = QueryBuilders.matchQuery("title", "小米");
    // 执行查询
    Iterable<Item> items = this.itemRepository.search(queryBuilder);
    items.forEach(System.out::println);
}
```

Repository的search方法需要QueryBuilder参数，elasticSearch为我们提供了一个对象QueryBuilders：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/37.png)

QueryBuilders提供了大量的静态方法，用于生成各种不同类型的查询对象，例如：词条、模糊、通配符等QueryBuilder对象。

结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/38.png)

elasticsearch提供很多可用的查询方式，但是不够灵活。如果想玩过滤或者聚合查询等就很难了。

#### 自定义查询

看看ElasticsearchRepository的底层先：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.springframework.data.elasticsearch.repository;

import java.io.Serializable;
import org.elasticsearch.index.query.QueryBuilder;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.query.SearchQuery;
import org.springframework.data.repository.NoRepositoryBean;

@NoRepositoryBean
public interface ElasticsearchRepository<T, ID extends Serializable> extends ElasticsearchCrudRepository<T, ID> {
    <S extends T> S index(S var1);

    Iterable<T> search(QueryBuilder var1);

    Page<T> search(QueryBuilder var1, Pageable var2);

    Page<T> search(SearchQuery var1);

    Page<T> searchSimilar(T var1, String[] var2, Pageable var3);

    void refresh();

    Class<T> getEntityClass();
}
```

先来看最基本的match query：

```java
@Test
public void testNativeQuery(){
    // 构建查询条件
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 添加基本的分词查询
    queryBuilder.withQuery(QueryBuilders.matchQuery("title", "小米"));
    // 执行搜索，获取结果
    Page<Item> items = this.itemRepository.search(queryBuilder.build());
    // 打印总条数
    System.out.println(items.getTotalElements());
    // 打印总页数
    System.out.println(items.getTotalPages());
    items.forEach(System.out::println);
}
```

`this.itemRepository.search()`中传入的是`queryBuilder.build()`，这里用到的就是上面底层代码中的第四个方法，`Page<T> search(SearchQuery var1);`，在这里，`queryBuilder.build();`正常返回的是`NativeSearchQuery`的对象，而`NativeSearchQuery`类正是`SearchQuery`接口的实现类。

NativeSearchQueryBuilder：Spring提供的一个查询条件构建器，帮助构建json格式的请求体。

`Page<item>`：默认是分页查询，因此返回的是一个分页的结果对象，包含属性：

- totalElements：总条数

- totalPages：总页数

- Iterator：迭代器，本身实现了Iterator接口，因此可直接迭代得到当前页的数据

- 其它属性：

  ![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/39.png)

结果

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/40.png)

#### 分页查询

利用`NativeSearchQueryBuilder`可以方便的实现分页：

```java
@Test
public void testNativeQuery(){
    // 构建查询条件
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 添加基本的分词查询
    queryBuilder.withQuery(QueryBuilders.termQuery("category", "手机"));

    // 初始化分页参数
    int page = 0;
    int size = 3;
    // 设置分页参数
    queryBuilder.withPageable(PageRequest.of(page, size));

    // 执行搜索，获取结果
    Page<Item> items = this.itemRepository.search(queryBuilder.build());
    // 打印总条数
    System.out.println(items.getTotalElements());
    // 打印总页数
    System.out.println(items.getTotalPages());
    // 每页大小
    System.out.println(items.getSize());
    // 当前页
    System.out.println(items.getNumber());
    items.forEach(System.out::println);
}
```

结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/41.png)

可以发现，`Elasticsearch中的分页是从第0页开始`。

#### 排序

排序也通用通过`NativeSearchQueryBuilder`完成：

```java
@Test
public void testSort(){
    // 构建查询条件
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 添加基本的分词查询
    queryBuilder.withQuery(QueryBuilders.termQuery("category", "手机"));

    // 排序
    queryBuilder.withSort(SortBuilders.fieldSort("price").order(SortOrder.DESC));

    // 执行搜索，获取结果
    Page<Item> items = this.itemRepository.search(queryBuilder.build());
    // 打印总条数
    System.out.println(items.getTotalElements());
    items.forEach(System.out::println);
}
```

结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/42.png)

### 聚合

#### 聚合为桶

桶就是分组，比如这里我们按照品牌brand进行分组：

```java
@Test
public void testAgg(){
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 不查询任何结果
    queryBuilder.withSourceFilter(new FetchSourceFilter(new String[]{""}, null));
    // 1、添加一个新的聚合，聚合类型为terms，聚合名称为brands，聚合字段为brand
    queryBuilder.addAggregation(
        AggregationBuilders.terms("brands").field("brand"));
    // 2、查询,需要把结果强转为AggregatedPage类型
    AggregatedPage<Item> aggPage = (AggregatedPage<Item>) this.itemRepository.search(queryBuilder.build());
    // 3、解析
    // 3.1、从结果中取出名为brands的那个聚合，
    // 因为是利用String类型字段来进行的term聚合，所以结果要强转为StringTerm类型
    StringTerms agg = (StringTerms) aggPage.getAggregation("brands");
    // 3.2、获取桶
    List<StringTerms.Bucket> buckets = agg.getBuckets();
    // 3.3、遍历
    for (StringTerms.Bucket bucket : buckets) {
        // 3.4、获取桶中的key，即品牌名称
        System.out.println(bucket.getKeyAsString());
        // 3.5、获取桶中的文档数量
        System.out.println(bucket.getDocCount());
    }

}
```

显示的结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/43.png)

关键API：

- `AggregationBuilders`：聚合的构建工厂类。所有聚合都由这个类来构建，看看他的静态方法：

  ![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/44.png)

- `AggregatedPage`：聚合查询的结果类。它是`Page<T>`的子接口：

  ![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/45.png)

  `AggregatedPage`在`Page`功能的基础上，拓展了与聚合相关的功能，它其实就是对聚合结果的一种封装，大家可以对照聚合结果的JSON结构来看。

  ![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/46.png)

  而返回的结果都是Aggregation类型对象，不过根据字段类型不同，又有不同的子类表示

  ![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/47.png)

我们看下页面的查询的JSON结果与Java类的对照关系：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/48.png)

#### 嵌套聚合，求平均值

代码：

```java
@Test
public void testSubAgg(){
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 不查询任何结果
    queryBuilder.withSourceFilter(new FetchSourceFilter(new String[]{""}, null));
    // 1、添加一个新的聚合，聚合类型为terms，聚合名称为brands，聚合字段为brand
    queryBuilder.addAggregation(
        AggregationBuilders.terms("brands").field("brand")
        .subAggregation(AggregationBuilders.avg("priceAvg").field("price")) // 在品牌聚合桶内进行嵌套聚合，求平均值
    );
    // 2、查询,需要把结果强转为AggregatedPage类型
    AggregatedPage<Item> aggPage = (AggregatedPage<Item>) this.itemRepository.search(queryBuilder.build());
    // 3、解析
    // 3.1、从结果中取出名为brands的那个聚合，
    // 因为是利用String类型字段来进行的term聚合，所以结果要强转为StringTerm类型
    StringTerms agg = (StringTerms) aggPage.getAggregation("brands");
    // 3.2、获取桶
    List<StringTerms.Bucket> buckets = agg.getBuckets();
    // 3.3、遍历
    for (StringTerms.Bucket bucket : buckets) {
        // 3.4、获取桶中的key，即品牌名称  3.5、获取桶中的文档数量
        System.out.println(bucket.getKeyAsString() + "，共" + bucket.getDocCount() + "台");

        // 3.6.获取子聚合结果：
        InternalAvg avg = (InternalAvg) bucket.getAggregations().asMap().get("priceAvg");
        System.out.println("平均售价：" + avg.getValue());
    }

}
```

结果：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/es/49.png)





## 集群

单机的elasticsearch做数据存储，必然面临两个问题：海量数据存储问题、单点故障问题。

- 海量数据存储问题：将索引库从逻辑上拆分为N个分片（shard），存储到多个节点
- 单点故障问题：将分片数据在不同节点备份（replica ）

**ES集群相关概念**:

* 集群（cluster）：一组拥有共同的 cluster name 的 节点。

* <font color="red">节点（node)</font>   ：集群中的一个 Elasticearch 实例

* <font color="red">分片（shard）</font>：索引可以被拆分为不同的部分进行存储，称为分片。在集群环境下，一个索引的不同分片可以拆分到不同的节点中

  解决问题：数据量太大，单点存储量有限的问题。

  ![image-20200104124440086](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20200104124440086-5602723.png)

  > 此处，我们把数据分成3片：shard0、shard1、shard2

* 主分片（Primary shard）：相对于副本分片的定义。

* 副本分片（Replica shard）每个主分片可以有一个或者多个副本，数据和主分片一样。

  ​	

数据备份可以保证高可用，但是每个分片备份一份，所需要的节点数量就会翻一倍，成本实在是太高了！

为了在高可用和成本间寻求平衡，我们可以这样做：

- 首先对数据分片，存储到不同节点
- 然后对每个分片进行备份，放到对方节点，完成互相备份

这样可以大大减少所需要的服务节点数量，如图，我们以3分片，每个分片备份一份为例：

![image-20200104124551912](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20200104124551912.png)

现在，每个分片都有1个备份，存储在3个节点：

- node0：保存了分片0和1
- node1：保存了分片0和2
- node2：保存了分片1和2





### 搭建ES集群

参考：https://chanservy.github.io/posts/20220611/elasticsearch-install.html#%E9%83%A8%E7%BD%B2es%E9%9B%86%E7%BE%A4









### 集群脑裂问题



#### 集群职责划分

elasticsearch中集群节点有不同的职责划分：

![image-20210723223008967](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723223008967.png)



默认情况下，集群中的任何一个节点都同时具备上述四种角色。



但是真实的集群一定要将集群职责分离：

- master节点：对CPU要求高，但是内存要求第
- data节点：对CPU和内存要求都高
- coordinating节点：对网络带宽、CPU要求高

职责分离可以让我们根据不同节点的需求分配不同的硬件去部署。而且避免业务之间的互相干扰。

一个典型的es集群职责划分如图：

![image-20210723223629142](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723223629142.png)



#### 脑裂问题

脑裂是因为集群中的节点失联导致的。

例如一个集群中，主节点与其它节点失联：

![image-20210723223804995](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723223804995.png)

此时，node2和node3认为node1宕机，就会重新选主：

![image-20210723223845754](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723223845754.png)

当node3当选后，集群继续对外提供服务，node2和node3自成集群，node1自成集群，两个集群数据不同步，出现数据差异。

当网络恢复后，因为集群中有两个master节点，集群状态的不一致，出现脑裂的情况：

![image-20210723224000555](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723224000555.png)



解决脑裂的方案是，要求选票超过 ( eligible节点数量 + 1 ）/ 2 才能当选为主，因此eligible节点数量最好是奇数。对应配置项是discovery.zen.minimum_master_nodes，在es7.0以后，已经成为默认配置，因此一般不会发生脑裂问题



例如：3个节点形成的集群，选票必须超过 （3 + 1） / 2 ，也就是2票。node3得到node2和node3的选票，当选为主。node1只有自己1票，没有当选。集群中依然只有1个主节点，没有出现脑裂。



#### 小结

master eligible节点的作用是什么？

- 参与集群选主
- 主节点可以管理集群状态、管理分片信息、处理创建和删除索引库的请求

data节点的作用是什么？

- 数据的CRUD

coordinator节点的作用是什么？

- 路由请求到其它节点

- 合并查询到的结果，返回给用户





### 集群分布式存储

当新增文档时，应该保存到不同分片，保证数据均衡，那么coordinating node如何确定数据该存储到哪个分片呢？



#### 分片存储测试

插入三条数据：

![image-20210723225006058](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225006058.png)



![image-20210723225034637](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225034637.png)



![image-20210723225112029](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225112029.png)



测试可以看到，三条数据分别在不同分片：

![image-20210723225227928](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225227928.png)

结果：

![image-20210723225342120](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225342120.png)



#### 分片存储原理



elasticsearch会通过hash算法来计算文档应该存储到哪个分片：

![image-20210723224354904](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723224354904.png)



说明：

- _routing默认是文档的id
- 算法与分片数量有关，因此索引库一旦创建，分片数量不能修改！





新增文档的流程如下：

![image-20210723225436084](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225436084.png)



解读：

- 1）新增一个id=1的文档
- 2）对id做hash运算，假如得到的是2，则应该存储到shard-2
- 3）shard-2的主分片在node3节点，将数据路由到node3
- 4）保存文档
- 5）同步给shard-2的副本replica-2，在node2节点
- 6）返回结果给coordinating-node节点



### 集群分布式查询

elasticsearch的查询分成两个阶段：

- scatter phase：分散阶段，coordinating node会把请求分发到每一个分片

- gather phase：聚集阶段，coordinating node汇总data node的搜索结果，并处理为最终结果集返回给用户



![image-20210723225809848](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225809848.png)









### 集群故障转移

集群的master节点会监控集群中的节点状态，如果发现有节点宕机，会立即将宕机节点的分片数据迁移到其它节点，确保数据安全，这个叫做故障转移。



1）例如一个集群结构如图：

![image-20210723225945963](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723225945963.png)

现在，node1是主节点，其它两个节点是从节点。



2）突然，node1发生了故障：

![image-20210723230020574](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723230020574.png)



宕机后的第一件事，需要重新选主，例如选中了node2：

![image-20210723230055974](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723230055974.png)



node2成为主节点后，会检测集群监控状态，发现：shard-1、shard-0没有副本节点。因此需要将node1上的数据迁移到node2、node3：

![image-20210723230216642](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210723230216642.png)



博客中的案例代码：

https://github.com/ChanServy/hotel_admin

https://github.com/ChanServy/hotel_demo







Over!!!



