---
title: Redis基础命令用法
date: 2022-04-03 22:08:12
categories: Redis
tags: Redis
urlname: redis-basics
---

## 基本指令

### 操作指令

读写数据：

* 设置 key，value 数据：

  ```sh
  set key value
  #set name seazean
  ```

* 根据 key 查询对应的 value，如果**不存在，返回空（nil）**：

  ```sh
  get key
  #get name
  ```

<!--more-->

帮助信息：

* 获取命令帮助文档

  ```sh
  help [command]
  #help set
  ```

* 获取组中所有命令信息名称

  ```sh
  help [@group-name]
  #help @string
  ```

退出服务

* 退出客户端：

  ```sh
  quit
  exit
  ```

* 退出客户端服务器快捷键：

  ```sh
  Ctrl+C
  ```


### key 指令

key 是一个字符串，通过 key 获取 redis 中保存的数据

* 基本操作

  ```sh
  del key                     #删除指定key
  unlink key                  #非阻塞删除key，真正的删除会在后续异步操作
  exists key                  #获取key是否存在
  type key                    #获取key的类型
  sort key [ASC/DESC]         #对key中数据排序，默认对数字排序，并不更改集合中的数据位置，只是查询
  sort key alpha              #对key中字母排序
  rename key newkey           #改名
  renamenx key newkey         #改名
  ```

* 时效性控制

  ```sh
  expire key seconds          #为指定key设置有效期，单位为秒
  pexpire key milliseconds    #为指定key设置有效期，单位为毫秒
  expireat key timestamp      #为指定key设置有效期，单位为时间戳
  pexpireat key mil-timestamp #为指定key设置有效期，单位为毫秒时间戳
  
  ttl key                     #获取key的有效时间，每次获取会自动变化(减小)，类似于倒计时，
                                #-1代表永久性，-2代表不存在/失效
  pttl key                    #获取key的有效时间，单位是毫秒，每次获取会自动变化(减小)
  persist key                 #切换key从时效性转换为永久性
  ```

* 查询模式

  ```sh
  keys pattern                #查询key
  ```

  查询模式规则：*匹配任意数量的任意符号；?配合一个任意符号；[]匹配一个指定符号

  ```sh
  keys *                      #查询所有key
  keys aa*                    #查询所有以aa开头
  keys *bb                    #查询所有以bb结尾
  keys ??cc                   #查询所有前面两个字符任意，后面以cc结尾 
  keys user:?                 #查询所有以user:开头，最后一个字符任意
  keys u[st]er:1              #查询所有以u开头，以er:1结尾，中间包含一个字母，s或t
  ```


### DB 指令

Redis 在使用过程中，随着操作数据量的增加，会出现大量的数据以及对应的 key，数据不区分种类、类别混在一起，容易引起重复或者冲突，所以 Redis 为每个服务提供 16 个数据库，编码 0-15，每个数据库之间相互独立，**共用 **Redis 内存，不区分大小

* 基本操作

  ```sh
  select index    #切换数据库，index从0-15取值
  ping            #测试数据库是否连接正常，返回PONG
  echo message    #控制台输出信息
  ```

* 扩展操作

  ```sh
  move key db     #数据移动到指定数据库，db是数据库编号
  dbsize          #获取当前数据库的数据总量，即key的个数
  flushdb         #清除当前数据库的所有数据
  flushall        #清除所有数据
  ```

### 通信指令

Redis 发布订阅（pub/sub）是一种消息通信模式：发送者（pub）发送消息，订阅者（sub）接收消息。

Redis 客户端可以订阅任意数量的频道

操作命令：

1. 打开一个客户端订阅 channel1：`SUBSCRIBE channel1`
2. 打开另一个客户端，给 channel1发布消息 hello：`publish channel1 hello`
3. 第一个客户端可以看到发送的消息

注意：发布的消息没有持久化，所以订阅的客户端只能收到订阅后发布的消息

### ACL 指令

Redis ACL 是 Access Control List（访问控制列表）的缩写，该功能允许根据可以执行的命令和可以访问的键来限制某些连接

* acl cat：查看添加权限指令类别
* acl whoami：查看当前用户

* acl setuser username on >password ~cached:* +get：设置有用户名、密码、ACL 权限（只能 get）

## 数据类型

注：Redis 中存储的是 key-value 形式的数据，并且 Redis 中的 key 永远都是 String 类型的，因此以下的几种数据类型都是针对 value 的！不要搞混了！

### String

#### 简介

String类型，也就是字符串类型，是Redis中最简单的存储类型。其value是字符串，不过根据字符串的格式不同，又可以分为3类：

- string：普通字符串
- int：整数类型，可以做自增、自减操作
- float：浮点类型，可以做自增、自减操作

不管是哪种格式，底层都是字节数组形式存储，只不过是编码方式不同。字符串类型的最大空间不能超过512m。

1. 存储的数据：单个数据，是最简单的数据存储类型，也是最常用的数据存储类型，实质上是存一个字符串。string 类型是二进制安全的，意味着 Redis 的 string 可以包含任何数据，比如图片或者序列化的对象
2. 存储数据的格式：一个存储空间保存一个数据，每一个空间中只能保存一个字符串信息
3. 存储内容：通常使用字符串，如果字符串以整数的形式展示，可以作为数字操作使用

Redis 所有操作都是**原子性**的，采用**单线程**机制，命令是单个顺序执行，无需考虑并发带来影响，原子性就是有一个失败则都失败。

#### 操作

指令操作：

* 数据操作：

  ```sh
  set key value           #添加/修改数据添加/修改数据
  del key                 #删除数据
  setnx key value         #判定性添加数据，键值为空则添加，如果这个键原先存在就不会添加、不会覆盖
  mset k1 v1 k2 v2...     #添加/修改多个数据，m：Multiple
  append key value        #追加信息到原始信息后部（如果原始信息存在就追加，否则新建）
  ```

* 查询操作

  ```sh
  get key                 #获取数据
  mget key1 key2...       #获取多个数据
  strlen key              #获取数据字符个数（字符串长度）
  ```

* 设置数值数据增加/减少指定范围的值

  ```sh
  incr key                    #key++
  incrby key increment        #key+increment
  incrbyfloat key increment   #对小数操作
  decr key                    #key--
  decrby key increment        #key-increment
  ```

* 设置数据具有指定的生命周期

  ```sh
  setex key seconds value         #设置key-value存活时间，seconds单位是秒
  psetex key milliseconds value   #毫秒级
  ```

注意事项：

1. 数据操作不成功的反馈与数据正常操作之间的差异

   * 表示运行结果是否成功
     (integer) 0  → false                 失败

     (integer) 1  → true                  成功

   * 表示运行结果值
     (integer) 3  → 3                        3个

     (integer) 1  → 1                        1个

2. 数据未获取到时，对应的数据为（nil），等同于null

3. **数据最大存储量**：512MB

4. string 在 Redis 内部存储默认就是一个字符串，当遇到增减类操作 incr，decr 时**会转成数值型**进行计算

5. 按数值进行操作的数据，如果原始数据不能转成数值，或超越了Redis 数值上限范围，将报错
   9223372036854775807（java 中 Long 型数据最大值，Long.MAX_VALUE）

6. Redis 可用于控制数据库表主键 ID，为数据库表主键提供生成策略，保障数据库表的主键唯一性

#### 应用

主页高频访问信息显示控制，例如新浪微博大 V 主页显示粉丝数与微博数量

* 在 Redis 中为大 V 用户设定用户信息，以用户主键和属性值作为 key，后台设定定时刷新策略

  ```sh
  set user:id:3506728370:fans 12210947
  set user:id:3506728370:blogs 6164
  set user:id:3506728370:focuses 83
  ```

* 使用 JSON 格式保存数据

  ```sh
  user:id:3506728370 → {"fans":12210947,"blogs":6164,"focuses":83}
  ```

* key的设置约定：表名 : 主键名 : 主键值 : 字段名

  | 表名  | 主键名 | 主键值    | 字段名 |
  | ----- | ------ | --------- | ------ |
  | order | id     | 29437595  | name   |
  | equip | id     | 390472345 | type   |
  | news  | id     | 202004150 | title  |

#### 实现

Redis 字符串对象底层的数据结构实现主要是 int 和简单动态字符串 SDS，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用**预分配冗余空间**的方式来减少内存的频繁分配

```c
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度
     int len;
     //记录 buf 数组中未使用字节的数量
     int free;
     //字节数组，用于保存字符串
     char buf[];
}
```

内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度 len，当字符串长度小于 1M 时，扩容都是双倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间，需要注意的是字符串最大长度为 512M。

详解请参考文章：https://www.cnblogs.com/hunternet/p/9957913.html

### Hash

#### 简介

Hash 类型，也叫散列，类似于 Java 中的 HashMap 结构。String 结构是将对象序列化为 JSON 字符串后存储，当需要修改对象某个字段时很不方便。Hash 结构可以将对象中的每个字段独立存储，可以针对单个字段做 CRUD。

数据存储需求：对一系列存储的数据进行编组，方便管理，典型应用存储对象信息

数据存储结构：一个存储空间保存多个键值对数据

hash 类型：底层使用**哈希表**结构实现数据存储

Redis 中的 hash 整体类似于 Java 中的  `Map<String, Map<Object,object>>`，左边是 key，右边是值，中间叫 field 字段，本质上 **hash 存了一个 key-value 的存储空间**

hash 是指的一个数据类型，并不是一个数据

* 如果 field 数量较少，存储结构优化为**压缩列表结构**（有序）
* 如果 field 数量较多，存储结构使用 HashMap 结构（无序）

#### 操作

指令操作：

* 数据操作

  ```sh
  hset key field value        #添加/修改数据
  hdel key field1 [field2]    #删除数据，[]代表可选
  hsetnx key field value      #设置field的值，如果该field存在则不做任何操作
  hmset key f1 v1 f2 v2...    #添加/修改多个数据
  ```

* 查询操作

  ```sh
  hget key field              #获取指定field对应数据
  hgetall key                 #获取指定key所有数据
  hmget key field1 field2...  #获取多个数据
  hexists key field           #获取哈希表中是否存在指定的字段
  hlen key                    #获取哈希表中字段的数量
  ```

* 获取哈希表中所有的字段名或字段值

  ```sh
  hkeys key                   #获取所有的field	
  hvals key                   #获取所有的value
  ```

* 设置指定字段的数值数据增加指定范围的值

  ```sh
  hincrby key field increment      #指定字段的数值数据增加指定的值，increment为负数则减少
  hincrbyfloat key field increment #操作小数
  ```


注意事项

1. hash 类型中 value 只能存储字符串，不允许存储其他数据类型，不存在嵌套现象，如果数据未获取到，对应的值为（nil）
2. 每个 hash 可以存储 2^32 - 1 个键值对
3. hash 类型和对象的数据存储形式相似，并且可以灵活添加删除对象属性。但 hash 设计初衷不是为了存储大量对象而设计的，不可滥用，不可将 hash 作为对象列表使用
4. hgetall 操作可以获取全部属性，如果内部 field 过多，遍历整体数据效率就很会低，有可能成为数据访问瓶颈

```shell
127.0.0.1:6379> hset stu name chan
(integer) 1
127.0.0.1:6379> hset stu age 22
(integer) 1
127.0.0.1:6379> hset stu score 98
(integer) 1
127.0.0.1:6379> hgetall stu
1) "name"
2) "chan"
3) "age"
4) "22"
5) "score"
6) "98"
127.0.0.1:6379> hkeys stu
1) "name"
2) "age"
3) "score"
127.0.0.1:6379> hvals stu
1) "chan"
2) "22"
3) "98"
127.0.0.1:6379> hincrby stu age 2
(integer) 24
127.0.0.1:6379> hvals stu
1) "chan"
2) "24"
3) "98"
127.0.0.1:6379> hsetnx stu name siyu
(integer) 0
127.0.0.1:6379> hsetnx stu favorite game
(integer) 1
127.0.0.1:6379> hvals stu
1) "chan"
2) "24"
3) "98"
4) "game"
127.0.0.1:6379> hkeys stu
1) "name"
2) "age"
3) "score"
4) "favorite"
127.0.0.1:6379> 
```

#### 应用

```sh
user:id:3506728370 → {"name":"春晚","fans":12210862,"blogs":83}
```

对于以上数据，使用单条去存的话，存的条数会很多。但如果用 json 格式，存一条数据就够了。

假如现在只粉丝数量发生了变化，需要修改，使用 json 格式存储的话要把整个字符串替换，但是用单条存就不存在这个问题，只需要改其中粉丝数量就可以。

也可以实现购物车的功能，key 对应着每个用户，存储空间存储购物车的信息

#### 实现

##### 底层结构

哈希类型的内部编码有两种：ziplist（压缩列表）、hashtable（哈希表、字典）

当存储的数据量比较小的情况下，Redis 才使用压缩列表来实现字典类型，具体需要满足两个条件：

- 当键值对个数小于 hash-max-ziplist-entries 配置（默认 512 个）
- 每对键值都小于 hash-max-ziplist-value 配置（默认 64 字节）

ziplist 使用更加紧凑的结构实现多个元素的连续存储，所以在节省内存方面比 hashtable 更加优秀，当 ziplist 无法满足哈希类型时，Redis 会使用 hashtable 作为哈希的内部实现，因为数据量过大时 ziplist 的读写效率会下降，而 hashtable 的读写时间复杂度为 O(1)

##### 压缩列表

压缩列表（ziplist）是列表和哈希的底层实现之一，压缩列表用来紧凑数据存储，节省内存，有序。

压缩列表是由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结枃，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

##### 哈希表

Redis 字典使用散列表为底层实现，一个散列表里面有多个散列表节点，每个散列表节点就保存了字典中的一个键值对，发生哈希冲突采用**链表法**解决，存储无序。

* 为了避免散列表性能的下降，当装载因子大于 1 的时候，Redis 会触发扩容，将散列表扩大为原来大小的 2 倍左右
* 当数据动态减少之后，为了节省内存，当装载因子小于 0.1 的时候，Redis 就会触发缩容，缩小为字典中数据个数的 50% 左右

### List

#### 简介

Redis 中的 List 类型与 Java 中的 LinkedList 类似，可以看做是一个双向链表结构。既可以支持正向检索和也可以支持反向检索。

特征也与 LinkedList 类似：

- 有序
- 元素可以重复
- 插入和删除快
- 查询速度一般

常用来存储一个**有序**数据，例如：朋友圈点赞列表，评论列表等。

数据存储需求：存储多个数据，并对数据进入存储空间的顺序进行区分

数据存储结构：一个存储空间保存多个数据，且通过数据可以体现进入顺序，允许重复元素

list 类型：保存多个数据，底层使用**双向链表**存储结构实现，类似于 LinkedList

如果两端都能存取数据的话，这就是双端队列，如果只能从同一端进同一端出，这个模型叫栈

#### 操作

指令操作：

* 数据操作

  ```sh
  lpush key value1 [value2]...  #从左边添加/修改数据
  rpush key value1 [value2]...  #从右边添加/修改数据
  lpop key                      #从左边获取并移除第一个数据，类似于出栈/出队，没有则返回nil
  rpop key                      #从右边获取并移除第一个数据，没有则返回nil
  lrem key count value          #删除指定数据，count=2删除2个，该value可能有多个(重复数据)
  ```

* 查询操作

  ```sh
  lrange key start stop       #从左边遍历数据并指定开始和结束索引，0是第一个索引，-1是终索引
  lindex key index            #获取指定索引数据，没有则为nil，没有索引则越界
  llen key                    #list中数据长度/个数
  ```

* 规定时间内获取并移除数据

  ```sh
  b                           #b代表阻塞
  blpop key1 [key2] timeout   #与LPOP和RPOP类似，只不过在没有元素时等待指定的时间，而不是直接返回nil，超时则为(nil)
                              #可以从其他客户端写数据，当前客户端阻塞读取数据
  brpop key1 [key2] timeout   #从右边操作
  ```

* 复制操作

  ```sh
  brpoplpush source destination timeout    #从source获取数据放入destination，假如在指定时间内没有任何元素被弹出，则返回一个nil和等待时长。反之，返回一个含有两个元素的列表，第一个元素是被弹出元素的值，第二个元素是等待时长
  ```

注意事项

1. list 中保存的数据都是 string 类型的，数据总容量是有限的，最多 2^32 - 1 个元素（4294967295）
2. list 具有索引的概念，但操作数据时通常以队列的形式进行入队出队，或以栈的形式进行入栈出栈
3. 获取全部数据操作结束索引设置为 -1
4. list 可以对数据进行分页操作，通常第一页的信息来自于 list，第 2 页及更多的信息通过数据库的形式加载

练习：

```shell
127.0.0.1:6379> lpush names zhangsan lisi
(integer) 2
127.0.0.1:6379> keys *
1) "name"
2) "k2"
3) "names"
4) "stu"
5) "user:1"
6) "user:2"
127.0.0.1:6379> get names
(error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> lrange names 0 -1
1) "lisi"
2) "zhangsan"
127.0.0.1:6379> lpush names chan tom
(integer) 4
127.0.0.1:6379> lrange names 0 -1
1) "tom"
2) "chan"
3) "lisi"
4) "zhangsan"
127.0.0.1:6379> rpush names jerry
(integer) 5
127.0.0.1:6379> lrange names 0 -1
1) "tom"
2) "chan"
3) "lisi"
4) "zhangsan"
5) "jerry"
127.0.0.1:6379> lpop names
"tom"
127.0.0.1:6379> lrange names 0 -1
1) "chan"
2) "lisi"
3) "zhangsan"
4) "jerry"
127.0.0.1:6379> rpop names
"jerry"
127.0.0.1:6379> lrange names 0 -1
1) "chan"
2) "lisi"
3) "zhangsan"
127.0.0.1:6379> brpop names 5
1) "names"
2) "zhangsan"
127.0.0.1:6379> lrange names 0 -1
1) "chan"
2) "lisi"
127.0.0.1:6379> brpop names 5
1) "names"
2) "lisi"
127.0.0.1:6379> lrange names 0 -1
1) "chan"
127.0.0.1:6379> brpop names 5
1) "names"
2) "chan"
127.0.0.1:6379> lrange names 0 -1
(empty array)
127.0.0.1:6379> brpop names 5
(nil)
(5.02s)
127.0.0.1:6379> 
```

#### 应用

企业运营过程中，系统将产生出大量的运营数据，如何保障多台服务器操作日志的统一顺序输出？

* 依赖 list 的数据具有顺序的特征对信息进行管理，右进左查或者左近左查
* 使用队列模型解决多路信息汇总合并的问题
* 使用栈模型解决最新消息的问题

微信文章订阅公众号：

* 比如订阅了两个公众号，它们发布了两篇文章，文章 ID 分别为 666 和 888，可以通过执行 `LPUSH key 666 888` 命令推送给我

#### 实现

##### 底层结构

在 Redis3.2 版本以前列表类型的内部编码有两种：ziplist（压缩列表）和 linkedlist（链表）

列表中存储的数据量比较小的时候，列表就会使用一块连续的内存存储，采用压缩列表的方式实现：

* 列表中保存的单个数据（有可能是字符串类型的）小于 64 字节
* 列表中数据个数少于 512 个

在 Redis3.2 版本以后对列表数据结构进行了改造，使用 **quicklist（快速列表）**代替了 linkedlist

##### 链表结构

Redis 链表为**双向无环链表**，使用 listNode 结构表示

```c
typedef struct listNode
{ 
	// 前置节点 
	struct listNode *prev; 
	// 后置节点 
	struct listNode *next; 
	// 节点的值 
	void *value; 
} listNode;
```

- 双向：链表节点带有前驱、后继指针，获取某个节点的前驱、后继节点的时间复杂度为 O(1)
- 无环：链表为非循环链表，表头节点的前驱指针和表尾节点的后继指针都指向 NULL，对链表的访问以 NULL 为终点

##### 快速列表

quicklist 实际上是 ziplist 和 linkedlist 的混合体，将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来，既满足了快速的插入删除性能，又不会出现太大的空间冗余

### Set

#### 简介

Redis 的 Set 结构与 Java 中的 HashSet 类似，可以看做是一个 value 为 null 的 HashMap。因为也是一个 hash 表，因此具备与 HashSet 类似的特征：

- 无序
- 元素不可重复
- 查找快
- 支持交集、并集、差集等功能

数据存储需求：存储大量的数据，在查询方面提供更高的效率

数据存储结构：能够保存大量的数据，高效的内部存储机制，便于查询

set 类型：与 hash 存储结构哈希表完全相同，只是仅存储键不存储值（nil），所以添加，删除，查找的复杂度都是 O(1)，并且**值是不允许重复且无序的**。

#### 操作

指令操作：

* 数据操作

  ```sh
  sadd key member1 [member2]  #添加数据
  srem key member1 [member2]  #删除数据
  ```

* 查询操作

  ```sh
  smembers key                #获取全部数据
  scard key                   #获取集合数据总量，返回set中数据的个数
  sismember key member        #判断集合中是否包含指定数据
  ```

* 随机操作

  ```sh
  spop key [count]            #随机获取集中的某个数据并将该数据移除集合
  srandmember key [count]     #随机获取集合中指定(数量)的数据
  ```

* 集合的交、并、差

  ```sh
  sinter key1 [key2...]                   #两个集合的交集，不存在为(empty list or set)
  sunion key1 [key2...]                   #两个集合的并集
  sdiff key1 [key2...]                    #两个集合的差集
  
  sinterstore destination key1 [key2...]  #两个集合的交集并存储到指定集合中
  sunionstore destination key1 [key2...]  #两个集合的并集并存储到指定集合中
  sdiffstore destination key1 [key2...]   #两个集合的差集并存储到指定集合中
  ```

* 复制

  ```sh
  smove source destination member         #将指定数据从原始集合中移动到目标集合中
  ```


注意事项

1. set 类型不允许数据重复，如果添加的数据在 set 中已经存在，将只保留一份
2. set 虽然与 hash 的存储结构相同，但是无法启用 hash 中存储值的空间

练习：

将下列数据用Redis的Set集合来存储：

张三的好友有：李四、王五、赵六；李四的好友有：王五、麻子、二狗

利用Set的命令实现下列功能：

- 计算张三的好友有几人
- 计算张三和李四有哪些共同好友
- 查询哪些人是张三的好友却不是李四的好友
- 查询张三和李四的好友总共有哪些人
- 判断李四是否是张三的好友
- 判断张三是否是李四的好友
- 将李四从张三的好友列表中移除

```shell
127.0.0.1:6379> sadd zs lisi wangwu zhaoliu
(integer) 3
127.0.0.1:6379> sadd ls wangwu mazi ergou
(integer) 3
127.0.0.1:6379> scard zs
(integer) 3
127.0.0.1:6379> sinter zs ls
1) "wangwu"
127.0.0.1:6379> sdiff zs ls
1) "zhaoliu"
2) "lisi"
127.0.0.1:6379> smembers ls
1) "ergou"
2) "mazi"
3) "wangwu"
127.0.0.1:6379> smembers zs
1) "zhaoliu"
2) "lisi"
3) "wangwu"
127.0.0.1:6379> sunion zs ls
1) "lisi"
2) "wangwu"
3) "ergou"
4) "zhaoliu"
5) "mazi"
127.0.0.1:6379> sismember zs lisi
(integer) 1
127.0.0.1:6379> sismember ls zhangsan
(integer) 0
127.0.0.1:6379> smembers zs
1) "zhaoliu"
2) "lisi"
3) "wangwu"
127.0.0.1:6379> smembers ls
1) "ergou"
2) "mazi"
3) "wangwu"
127.0.0.1:6379> srem zs lisi
(integer) 1
127.0.0.1:6379> smembers ls
1) "ergou"
2) "mazi"
3) "wangwu"
127.0.0.1:6379> smembers zs
1) "zhaoliu"
2) "wangwu"
127.0.0.1:6379> 
```

#### 应用

应用场景：

1. 黑名单：资讯类信息类网站追求高访问量，但是由于其信息的价值，往往容易被不法分子利用，通过爬虫技术，快速获取信息，个别特种行业网站信息通过爬虫获取分析后，可以转换成商业机密。

   注意：爬虫不一定做摧毁性的工作，有些小型网站需要爬虫为其带来一些流量。

2. 白名单：对于安全性更高的应用访问，仅仅靠黑名单是不能解决安全问题的，此时需要设定可访问的用户群体， 依赖白名单做更为苛刻的访问验证

3. 随机操作可以实现抽奖功能

4. 集合的交并补可以实现微博共同关注的查看，可以根据共同关注或者共同喜欢推荐相关内容

#### 实现

集合类型的内部编码有两种：

* intset（整数集合）：当集合中的元素都是整数且元素个数小于 set-maxintset-entries配置（默认 512 个）时，Redis 会选用 intset 来作为集合的内部实现，从而减少内存的使用

* hashtable（哈希表，字典）：当无法满足 intset 条件时，Redis 会使用 hashtable 作为集合的内部实现

整数集合（intset）是 Redis 用于保存整数值的集合抽象数据结构，可以保存类型为 int16_t、int32_t 或者 int64_t 的整数值，并且保证集合中的元素是**有序不重复**的

### SortedSet

#### 简介

Redis 的 SortedSet 是一个可排序的 set 集合，与 Java 中的 TreeSet 有些类似，但底层数据结构却差别很大。SortedSet 中的每一个元素都带有一个 score 属性（比如分数、票数等业务参数），可以基于 score 属性对元素排序，底层的实现是一个跳表（SkipList）加 hash 表。

SortedSet 具备下列特性：

- 可排序
- 元素不重复
- 查询速度快

因为 SortedSet 的可排序特性，经常被用来实现排行榜这样的功能。

数据存储需求：数据排序有利于数据的有效展示，需要提供一种可以根据自身特征进行排序的方式

数据存储结构：新的存储模型，可以保存可排序的数据

sorted_set 类型：在 set 的存储结构基础上添加可排序字段，类似于 TreeSet

#### 操作

指令操作：

* 数据操作

  ```sh
  zadd key score1 member1 [score2 member2]    #添加数据
  zrem key member [member ...]                #删除数据
  zremrangebyrank key start stop              #删除指定索引范围的数据
  zremrangebyscore key min max                #删除指定分数区间内的数据
  zscore key member                           #获取指定值的分数
  zincrby key increment member                #指定值的分数增加increment
  ```

* 查询操作

  ```sh
  zrange key start stop [WITHSCORES]      #获取指定范围的数据，升序，WITHSCORES 代表显示分数
  zrevrange key start stop [WITHSCORES]   #获取指定范围的数据，降序
  
  zrangebyscore key min max [WITHSCORES] [LIMIT offset count]  #按条件获取数据，从小到大
  zrevrangebyscore key max min [WITHSCORES] [...]              #从大到小
  
  zcard key                                       #获取集合数据的总量
  zcount key min max                              #获取指定分数区间内的数据总量
  zrank key member                                #获取数据对应的索引（排名）升序
  zrevrank key member                             #获取数据对应的索引（排名）降序
  ```

  * min 与 max 用于限定搜索查询的条件
  * start 与 stop 用于限定查询范围，作用于索引，表示开始和结束索引
  * offset 与 count 用于限定查询范围，作用于查询结果，表示开始位置和数据总量

* 集合的交、并操作

  ```sh
  zinterstore destination numkeys key [key ...]   #两个集合的交集并存储到指定集合中
  zunionstore destination numkeys key [key ...]   #两个集合的并集并存储到指定集合中
  ```

注意事项：

1. score 保存的数据存储空间是 64 位，如果是整数范围是 -9007199254740992~9007199254740992
2. score 保存的数据也可以是一个双精度的 double 值，基于双精度浮点数的特征可能会丢失精度，慎重使用
3. sorted_set 底层存储还是基于 set 结构的，因此数据不能重复，如果重复添加相同的数据，score 值将被反复覆盖，保留最后一次修改的结果。

练习：

将班级的下列学生得分存入Redis的SortedSet中：Jack 85, Lucy 89, Rose 82, Tom 95, Jerry 78, Amy 92, Miles 76，并实现下列功能：

- 删除Tom同学
- 获取Amy同学的分数
- 获取Rose同学的排名
- 查询80分以下有几个学生
- 给Amy同学加2分
- 查出成绩前3名的同学
- 查出成绩80分以下的所有同学

```shell
127.0.0.1:6379> zadd exam 85 jack 89 lucy 82 rose 95 tom 78 jerry 92 amy 76 miles
(integer) 7
127.0.0.1:6379> zrank exam jack
(integer) 3
127.0.0.1:6379> zrem exam tom
(integer) 1
127.0.0.1:6379> zscore exam amy
"92"
127.0.0.1:6379> zrank exam rose
(integer) 2
127.0.0.1:6379> zrevrank exam rose
(integer) 3
127.0.0.1:6379> zcount exam 0 80
(integer) 2
127.0.0.1:6379> zincrby exam 2 amy
"94"
127.0.0.1:6379> zrevrange exam 0 2
1) "amy"
2) "lucy"
3) "jack"
127.0.0.1:6379> zrangebyscore exam 0 80
1) "miles"
2) "jerry"
127.0.0.1:6379> 
```

#### 应用

* 排行榜
* 对于基于时间线限定的任务处理，将处理时间记录为 score 值，利用排序功能区分处理的先后顺序
* 当任务或者消息待处理，形成了任务队列或消息队列时，对于高优先级的任务要保障对其优先处理，采用 score 记录权重

#### 实现

##### 底层结构

有序集合是由 ziplist（压缩列表）或 skiplist（跳跃表）组成的

当数据比较少时，有序集合使用的是 ziplist 存储的，使用 ziplist 格式存储需要满足以下两个条件：

- 有序集合保存的元素个数要小于 128 个；
- 有序集合保存的所有元素大小都小于 64 字节

当元素比较多时，此时 ziplist 的读写效率会下降，时间复杂度是 O(n)，跳表的时间复杂度是 O(logn)

##### 跳跃表

Redis 使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的**元素数量比较多**，又或者有序集合中元素的**成员是比较长的字符串**时，Redis 就会使用跳跃表来作为有序集合健的底层实现

跳跃表在链表的基础上增加了多级索引以提升查找的效率，索引是占内存的，所以是一个**空间换时间**的方案。原始链表中存储的有可能是很大的对象，而索引结点只需要存储关键值和几个指针，并不需要存储对象，因此当节点本身比较大或者元素数量比较多的时候，其优势可以被放大，而缺点则可以忽略

* 基于单向链表加索引的方式实现

- Redis 的跳跃表实现由 zskiplist 和 zskiplistnode 两个结构组成，其中 zskiplist 用于保存跳跃表信息（比如表头节点、表尾节点、长度），而 zskiplistnode 则用于表示跳跃表节点
- Redis 每个跳跃表节点的层高都是 1 至 32 之间的随机数（Redis5 之后最大层数为 64）
- 在同一个跳跃表中，多个节点可以包含相同的分值，但每个节点的成员对象必须是唯一的。跳跃表中的节点按照分值大小进行排序，当分值相同时节点按照成员对象的大小进行排序

个人笔记：JUC → 并发包 → ConcurrentSkipListMap 详解跳跃表

参考文章：https://www.cnblogs.com/hunternet/p/11248192.html

### BitMap

#### 操作

BitMap 本身不是一种数据类型， 实际上就是字符串（key-value） ， 但是它可以对字符串的位进行操作

数据结构的详解查看 Java → Algorithm → 位图

指令操作：

* 获取指定 key 对应**偏移量**上的 bit 值

  ```sh
  getbit key offset
  ```

* 设置指定 key 对应偏移量上的 bit 值，value 只能是 1 或 0

  ```sh
  setbit key offset value
  ```

* 对指定 key 按位进行交、并、非、异或操作，并将结果保存到 destKey 中

  ```sh
  bitop option destKey key1 [key2...]
  ```

  option：and 交、or 并、not 非、xor 异或

* 统计指定 key 中1的数量

  ```sh
  bitcount key [start end]
  ```

#### 应用

- 解决 Redis 缓存穿透，判断给定数据是否存在， 防止缓存穿透

- 垃圾邮件过滤，对每一个发送邮件的地址进行判断是否在布隆的黑名单中，如果在就判断为垃圾邮件

- 爬虫去重，爬给定网址的时候对已经爬取过的 URL 去重

- 信息状态统计

### HyperLog

基数是数据集去重后元素个数，HyperLogLog 是用来做基数统计的，运用了 LogLog 的算法

```java
{1, 3, 5, 7, 5, 7, 8} 	基数集： {1, 3, 5 ,7, 8} 	基数：5
{1, 1, 1, 1, 1, 7, 1} 	基数集： {1,7} 				基数：2
```

相关指令：

* 添加数据

  ```sh
  pfadd key element [element ...]
  ```

* 统计数据

  ```sh
  pfcount key [key ...]
  ```

* 合并数据

  ```sh
  pfmerge destkey sourcekey [sourcekey...]
  ```

应用场景：

* 用于进行基数统计，不是集合不保存数据，只记录数量而不是具体数据，比如网站的访问量
* 核心是基数估算算法，最终数值存在一定误差
* 误差范围：基数估计的结果是一个带有 0.81% 标准错误的近似值
* 耗空间极小，每个 hyperloglog key 占用了12K的内存用于标记基数
* pfadd 命令不是一次性分配12K内存使用，会随着基数的增加内存逐渐增大
* Pfmerge 命令合并后占用的存储空间为12K，无论合并之前数据量多少

### GEO

GeoHash 是一种地址编码方法，把二维的空间经纬度数据编码成一个字符串

* 添加坐标点

  ```sh
  geoadd key longitude latitude member [longitude latitude member ...]
  georadius key longitude latitude radius m|km|ft|mi [withcoord] [withdist] [withhash] [count count]
  ```

* 获取坐标点

  ```sh
  geopos key member [member ...]
  georadiusbymember key member radius m|km|ft|mi [withcoord] [withdist] [withhash] [count count]
  ```

* 计算距离

  ```sh
  geodist key member1 member2 [unit]  #计算坐标点距离
  geohash key member [member ...]     #计算经纬度
  ```

redis 应用于地理位置计算。
