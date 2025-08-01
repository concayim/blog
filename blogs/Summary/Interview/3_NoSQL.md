---
title: NoSQL专题
tags: 
  - 面试
date: 2023-05-17 09:00:00
categories:
  - 面试
---

## NoSQL专题

### redis数据结构

拼多多

1、Redis都有哪些数据类型?简述一下SortedSet底层原理? 

字节

1、redis的数据结构dict，如何rehash

2、zset的数据结构跳表，如何插入

3、redis的字符串实现结构

#### 题解

##### dict

redis是一个支持key-value内存存储的数据结构服务器，如下

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221538806.png)

Redis的dict实现最显著的一个特点，就在于它的重哈希。它采用了一种称为增量式重哈希(incremental rehashing，也叫渐进式重哈希的方法， 在需要扩展内存时避免一次性对所有key进行重哈希，而是将重哈希操作分散到对于dict的各个增删改查的操作中去。这种方法能做到每次只对一小部分key进行重哈希，而每次重哈希之间不影响dict的操作

如下图

![image-20230522153905848](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221539877.png)

##### robj

redis的一个database内的这个映射关系是用一个 dict 来维护的，为了在 同一个 dict 内能够存储不同类型的value，这就需要一个通用的数据结构，这 个通用的数据结构就是robj(全名redisObject)

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221540192.png)

一个robj包含如下5个字段，图示如下

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221544265.png)

这里特别需要仔细察看的是encoding字段。对于同一个type，还可能对应不同的encoding，这说明同样的一个数据类型，可能存在不同的内部表示方式。而不同的内部表示，在内存占用和查找性能上会有所不同，下图展示了redis 5种常见数据类型在不同编码方式下的数据结构

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221545128.png)

##### sds

Redis中的字符串是一种动态字符串，即Simple Dynamic String(SDS),这意味着使用者可以修改，它的底层实现有点类似于Java中的ArrayList。

SDS 结构如图所示

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221548654.png)

注意：SDS在内存地址上是前后相邻的两部分组成的。

==SDS与C字符串==

1、C语言中是没有字符串类型的，字符串是存放在字符数组中的，java中的String内部也是构建的字符数组

2、C字符串获取字符串长度是O(N)的，而SDS是O(1)的，直接通过len属性获取，

3、SDS能很好的杜绝缓冲区溢出/内存泄漏 的问题。 因为C字符串不记录自身长度，如果执行拼接或者缩短字符串的操作，操作不当就很容易造成缓冲溢出/内存泄漏问题(比如忘记在操作之前分配足够的空间/用完了 忘记释放内存);

而SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性，当使用API对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的要求，如果不满足，它会自动将 SDS 的空间扩展至执行修改所需要的大小

为什么要区分embstr与raw编码？本质上是redis对内存的优化做到了极致，不同大小的字符串使用不同的sds的header，ebmstr和raw编码是以字符串长度是否在44字节内为分界的。即 embstr 编码形式，可以存储最大字符串长度是44字节。

==注：redis 3.0 版本之前是以 39字节为分界的==

然而embstr和raw的最大区别在于内存的分配上

embstr编码，代表string的robj和sds是一块连续的内存，只需分配一次内存

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221614923.png)

raw编码，代表string的robj和sds是两块内存，需要创建时需要进行两次内存分配，回收也是一样。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221615739.png)

##### skiplist

###### 经典skiplist

skiplist 本质上也是一种查找结构，用于解决算法中的查找问题 (Searching)，即根据给定的key，快速查到它所在的位置(即对应的 value)。

skiplist 是由William Pugh发明的，最早出现于他在1990年发表的论文 《Skip Lists: A Probabilistic Alternative to Balanced Trees》，对论文感兴趣的可下载查看。

因为 skiplist 是在链表的基础上发展起来的，所以先来看一个普通有序链表

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221627575.png)

我们要查找某条数据，做法只能是从头节点开始，依次遍历每个节点，比较是否是自己想找的数据，整体的时间复杂度是O(n)的，插入某条数据也是，需要先找到待插入的位置才能插入，那如何提高查找效率呢?

如果让每相邻两个节点间增加一个指针，让指针指向下下个节点，如下图

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221630465.png)

比如，我们想查找23

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221630465.png)

那如果按这种方式，我们再添加一层指针呢？

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221631449.png)

如果还想查找23，那么沿着最上层链表首先要比较的是19，发现23比19大，接下来我们就知道只需要到19的后面去继续查找，从而一下子跳过了19前面的所有节点。可以想象，当链表足够长的时候，这种多层链表的查找方式能让我们跳过很多下层节点，大大加快查找的速度。

skiplist正是受这种多层链表的想法的启发而设计出来的。实际上按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到O(log n)。

但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的 2:1 的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点(也包括新插入的节点) 重新进行调整，这会让时间复杂度重新蜕化成O(n)。删除数据也有同样的问题。

skiplist 为了避免这一问题，它不要求上下相邻两层链表之间的节点个数 有严格的对应关系，而是为每个节点随机出一个层数(level)。比如，一个节点随 机出的层数是3，那么就把它链入到第1层到第3层这三层链表中。为了表达清 楚，下图展示了如何通过一步步的插入操作从而形成一个skiplist的过程

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221647702.png)

每一个节点的层数( level )是随机出来的，而且新插入一个节点不会影响其它节点的层数。因此，插入操作只需要修改插入节点前后的指针，而不需要对很多节点都进行调整。这就降低了插入操作的复杂度。实际上，这是skiplist的一个很重要的特性，这让它在插入性能上明显优于平衡树的方案。

现在我们从跳表中查找23，查找过程如下

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221648820.png)

##### redis中的跳表

当数据多的时候，sorted set是由一个叫zset的数据结构来实现的，这个zset包含 dict + skiplist

dict 用来查询数据到分数( score )的对应关系，而skiplist用来根据分数查询数据(可能是范围查找)。

转化的条件是

```shell
zset-max-ziplist-entries 128 
zset-max-ziplist-value 64
```

- 当sorted set中的元素个数，即(数据, score)对的数目超过128的时候， 也就是ziplist数据项超过256的时候。
- 当sorted set中插入的任意一个数据的长度超过了64的时候。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221650495.png)

而 zskiplist 定义了真正的skiplist结构

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221650840.png)

---

### redis持久化

字节

1、redis持久化，rdb bgsave的时候存储的数据是拷贝给子进程的吗? 

2、Redis 持久化机制，AOF很大怎么办

微软

1、Redis 持久化的机制是什么? 

2、持久化的最佳实践是什么?

#### 题解

##### RDB快照

bgsave 通过fork一个子进程，专门用于写入 RDB 文件，避免了主线程的 阻塞，这也是 Redis RDB 文件生成的默认配，fork子进程负责持久化过程，阻 塞只会发生在fork子进程的时候。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221700680.png)

不会直接拷贝数据(内存使用率超过50%，直接拷贝不就gg了!!)

##### COW(Copy on Write):

fork一个子进程，只有在父进程发生写操作修改内存数据时，才会真正去分配内存空间，并复制内存数据，而且也只是复制被修改的内存页中的数据，并不是全部内存数据

fork采用操作系统提供的写实复制(Copy On Write)机制，就是为了避 免一次性拷贝大量内存数据给子进程造成的长时间阻塞问题，但fork子进程 需要拷贝进程必要的数据结构，其中有一项就是拷贝内存页表(虚拟内存和物理内存的映射索引表)，这个拷贝过程会消耗大量CPU资源，拷贝完成之前整个进程是会阻塞的，阻塞时间取决于整个实例的内存大小，实例越大，内存页表越大，fork阻塞时间越久。拷贝内存页表完成后，子进程与父进程指向相同的内存地址空间，也就是说此时虽然产生了子进程，但是并没有申请与父进程相同的内存大小。那什么时候父子进程才会真正内存分离呢？“写实复制”顾名思义，就是在写发生时，才真正拷贝内存真正的数据

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221707188.png)

##### AOF日志

AOF日志是写后日志，“写后”的意思是 Redis 是先执行命令，把数据写入内存，然后才记录日志，这种方式就是先让系统执行命令，只有命令能执行成功，才会被记录到日志中，否则系统就会直接向客户端报错。所以Redis使用写后日志这一方式的一大好处是，可以避免出现记录错误命令的情况

另外还有个好处是不会阻塞当前的写操作(但可能阻塞下一个操作)。

AOF 机制给我们提供了三个选择，也就是 AOF 配置项 appendfsync 的三个可选值。

1.Always，同步写回

每个写命令执行完，立马同步地将日志写回磁盘

2.Everysec，每秒写回

每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘

3.No，操作系统控制的写回

每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。

##### AOF文件过大怎么办? 

###### 重写

AOF 重写机制就是在重写时，Redis 根据数据库的现状创建一个新的 AOF 文件，也就是说，读取数据库中的所有键值对，然后对每一个键值对用一条命令记录它的写入。比如说，当读取了键值对“testkey”: “testvalue” 之后，重写机制会记录set testkey testvalue这条命令。这样，当需要恢复时，可以重新执行该命令，实现“testkey”: “testvalue”的写入

AOF 文件是以追加的方式，逐一记录接收到的写命令的。当一个键值对被多条写命令反复修改时，AOF 文件会记录相应的多条命令。但是在重写的时候，是根据这个键值对当前的最新状态，为它生成对应的写入命令。这样一来，一个键值对在重写日志中只用一条命令就行了，而且在日志恢复时，只用执行这条命令，就可以直接完成这个键值对的写入了。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221725178.png)

###### 原理

AOF 文件重写并不需要对现有的 AOF 文件进行任何读取、分析或者写入操作，而是通过读取服务器当前的数据库状态来实现的。首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键 值对的多条命令，这就是AOF重写功能的实现原理。

==一个拷贝，两处日志==

“一个拷贝”就是指，每次执行重写时，主线程fork出后台的bgrewriteaof子进程。此时fork会把主线程的内存拷贝一份给bgrewriteaof子进程(也是用到COW技术)，这里面就包含了数据库的最新数据。然后bgrewriteaof子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，记入重写日志。

“两处日志”又是什么呢?因为主线程未阻塞，仍然可以处理新来的操作。此时，如果有写操作，第一处日志就是指正在使用的AOF日志，Redis会把这个操作写到它的缓冲区。这样一来，即使宕机了，这个AOF日志的操作仍然是齐 全的，可以用于恢复。而第二处日志，就是指新的 AOF 重写日志。这个操作也 会被写到重写日志的缓冲区。这样，重写日志也不会丢失最新的操作。等到拷 贝数据的所有操作记录重写完成后，重写日志记录的这些最新操作也会写入新的AOF文件，以保证数据库最新状态的记录。此时，我们就可以用新的AOF文件替代旧文件了。

AOF非阻塞的重写过程总结来说，每次 AOF 重写时，Redis 会先执行一个内存拷贝，用于重写。然后使用两个日志保证在重写过程中，新写入的数据不会丢失。而且，因为 Redis 采用额外的线程进行数据重写，所以这个过程并不会阻塞主线程。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305230952553.png)

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305230952353.png)

##### 最佳实践

先谈数据恢复策略，如果一台服务器上有既有RDB文件，又有AOF文件，RDB和AOF对比

| -             | RDB                  | AOF          |
| ------------- | -------------------- | ------------ |
| 启动优先级    | 低                   | 高           |
| 体积          | 小                   | 大           |
| 恢复速度快 慢 | 快                   | 慢           |
| 数据安全性    | 丢失若干时间内的数据 | 根据策略决定 |

快照多久做一次?

-	时间太长，出现问题丢失的数据太多
-	时间太短，频繁写磁盘会带来压力，频繁fork子进程在fork时会阻塞主进程。

最好的方式:增量快照，一次全量快照后，记录增量的数据。

Redis 4.0中提出了一个混合使用AOF日志和内存快照的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用AOF日志记录这期间的所有命令操作。这样一来，快照不用很频繁地执行，这就避免了频繁fork对主线程的影响。而且AOF日志也只用记录两次快照间的操作，也就是说不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。

----

### redis删除策略

字节

1、Redis过期key怎么处理

阿里云

1、谈谈Redis的删除策略

#### 题解

##### 过期删除

在Redis内部，每当我们设置一个键的过期时间时，Redis就会将该键带上过期时间存放到一个过期字典中，对应三种策略

1、定时删除

在设置某个key 的过期时间同时，我们创建一个定时器，让定时器在该过期时间到来时，立即执行对其进行删除的操作。由于淘汰策略也是在主线程执行的，此种方案不推荐。

2、惰性删除

设置该key过期时间后，我们不去管它，当需要该key时，我们在检查其是否过期，如果过期，我们就删掉它，反之返回该key。Redis的惰性删除策略由 db.c/expireIfNeeded函数实现，所有键读写命令执行之前都会调用expireIfNeeded函数对其进行检查。

3、定期删除

每隔一段时间，我们就对一些key进行检查，删除里面过期的key。由redis.c/activeExpireCycle函数实现，函数以一定的频率运行，每次运行时都从一定数量的数据库中取出一定数量的随机键进行检查，并删除其中的过期键。

注意:并不是一次运行就检查所有的库，所有的键，而是随机检查一定数量的键。

定期删除函数的运行频率，在Redis2.6版本中，规定每秒运行10次，大概100ms运行一次。在Redis2.8版本后，可以通过修改配置文件redis.conf 的hz选项来调整这个次数

##### 最佳实践

==定期删除+惰性删除==

由于定期删除每次只会随机删除一部分，会滞留很多过期key在内存中，所以需要配合惰性删除在使用该key的时候去检测如果过期则删除。

##### 内存淘汰

虽然定期+惰性，但是如果定期删除漏掉了很多过期 key，然后你也没及时去查询，也就没走惰性删除，此时会怎么样?如果大量过期key堆积在内存里，导致 redis 内存块耗尽了，所以此时需要走内存淘汰机制，有如下几种淘汰方式

1.volatile-lru

设置了过期时间的key使用LRU算法淘汰

2.allkeys-lru

所有key使用LRU算法淘汰

3.volatile-lfu

设置了过期时间的key使用LFU算法淘汰

4.allkeys-lfu

所有key使用LFU算法淘汰

5.volatile-random

设置了过期时间的key使用随机淘汰

6.allkeys-random

所有key使用随机淘汰

7.volatile-ttl

设置了过期时间的key根据过期时间淘汰，越早过期越早淘汰

8.noeviction

默认策略，当内存达到设置的最大值时，所有申请内存的操作都会报错(如set,lpush等)，只读操作如get命令可以正常执行

##### LRU

表示最近最少使用，常见实现方式为链表:

新数据放在链表头部 ，链表中的数据被访问就移动到链头，链表满的时候从链表尾部移出数据。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231048967.png)

而在Redis中使用的是近似LRU算法，为什么说是近似呢?Redis中是随机 采样5个(可以修改参数maxmemory-samples配置)key，然后从中选择访问 时间最早的key进行淘汰，因此当采样key的数量与Redis库中key的数量越接 近，淘汰的规则就越接近LRU算法。但官方推荐5个就足够了，最多不超过10 个，越大就越消耗CPU的资源。

redis中对LRU并没有采用链表实现，因为它觉得太占用空间了

具体参看：https://www.php.cn/redis/459068.html

但在LRU算法下，如果一个热点数据最近很少访问，而非热点数据近期访问 了，就会误把热点数据淘汰而留下了非热点数据，因此在Redis4.x中新增了LFU 算法。

##### LFU

LFU(Least Frequently Used)表示最不经常使用，它是根据数据的历史访问频率来淘汰数据，其核心思想是“如果数据过去被访问多次，那么将来被访问的频率也更高”。

LFU算法反映了一个key的热度情况，不会因LRU算法的偶尔一次被访问被 误认为是热点数据。

LFU算法的常见实现方式为链表

新数据放在链表尾部 ，链表中的数据按照被访问次数降序排列，访问次数相同的按最近访问时间降序排列，链表满的时候从链表尾部移出数据。

---

### redis高可用 

字节

1、redis集群的模式，如何选主 2、redis的主从复制流程，主从复制会影响redis的功能吗

美团

1、redis集群的模式，如何选主

微软

1、master选举策略，数据流向。

#### 题解

##### 主从复制

当我们启动多个 Redis 实例的时候，它们相互之间就可以通过 replicaof(Redis 5.0 之前使用 slaveof)命令形成主库和从库的关系，之后会 按照三个阶段完成数据的第一次同步

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231101311.png)

一旦主从库完成了全量复制，它们之间就会一直维护一个网络连接，主库会通过这个连接将后续陆续收到的命令操作再同步给从库，这个过程也称为基于长连接的命令传播，可以避免频繁建立连接的开销。

主从之间网络断开后怎么办?

如果主从库在命令传播时出现了网络闪断，那么从库就会和主库重新进 行一次全量复制，开销非常大。

从 Redis 2.8 开始，网络断了之后，主从库会采 用增量复制的方式继续同步。全量复制是同步所有数据，而增量复制只会把主从库网络断连期间主库收到的命令，同步给从库。

当主从库断连后，主库会把断连期间收到的写操作命令，写入 replication buffer，同时也会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。repl_backlog_buffer 是一个环形缓冲区，主库会记录自己写到的位置，从库则 会记录自己已经读到的位置。

主从库的连接恢复之后，从库首先会给主库发送psync命令，并把自己当前的slave_repl_offset发给主库，主库会判断自己的master_repl_offset和 slave_repl_offset之间的差距。在网络断连阶段，主库可能会收到新的写操作命令，所以一般来说，master_repl_offset会大于slave_repl_offset。此时主库只用把master_repl_offset 和 slave_repl_offset之间的命令操作同步给从库就行。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231122489.png)

##### 集群故障转移

一个 Redis 集群是由多个 Redis 节点构成的，节点间可以相互通信。而每个节点又可以是多个 redis 服务器构成的主从模式。比如我们可以将九个节点，三三的构成主从模式(一主两从)，然后将三个主节点构成集群(三主六从)。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231123297.png)

##### 故障检测

集群中的每个节点都会保存一份集群里所有节点的状态表，集群中的每个节点都会定期的向其他节点发送PING消息，以此来检测对方是否下线。如果再规定的时间内没有收到PONG消息的响应，就会将该节点标记为疑似下线 (PFAIL)，集群中的各个节点会通过消息，来交换自己维护的节点状态表。 比如某个节点是在线、下线还是疑似下线。

如果集群里半数以上的主节点，都将某个主节点标记为疑似下线，那么这 个主节点将被标记为下线(FAIL)状态，将主节点标记为下线状态的节点将会向集群广播一条消息，所有收到消息的节点，都会将节点标记为下线状态。

##### leader选举

当从节点发现自己复制的主节点进入下线状态后，就会触发进入选举过程，redis集群将会在下线的主节点的从节点中选出一个节点作为新的主节点。

选举过程与哨兵选举过程非常类似。都是基于Raft算法做的。

首先，集群有一个配置纪元的计数器。初始值为0，每进行一次故障转移就会自增+1。集群中只有主节点有投票的权利，且每个主节点在一次故障转移 期间只有一次投票的机会。

当集群中的从节点发现自己复制的主节点下线后，就会广播一条消息，要求所有收到这条消息且具有投票权利的节点向自己投票。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231135425.png)

如果主节点还没有投票，则会返回 ACK 消息，表示支持该从节点成为主节 点。如果某个从节点收到的票数超过主节点个数的一半，那么这个从节点将成 为新的主节点。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305231135328.png)

如果集群中除下线节点外剩余的节点刚好能够被从节点瓜分，在极端情况 下就会出现所有从节点的票数都不会超过一半，就会造成本次选举失败，然后配置纪元+1，进行重新选举。比如上图中的例子，如果两个从节点同时发起投票，那么就有可能每个从节点都收到一票，导致本次选举失败。

redis集群为例尽量避免出现这种情况做了一下优化，在发现主节点下线后，从节点并不会立即发送消息，而是延迟一定的时间再发送选举消息，这个时间是随机的。计算公式如下

```
DELAY = 500 milliseconds
        + random delay between 0 and 500 milliseconds
        + SLAVE_RANK * 1000 milliseconds
```

500ms 加上一个随机时间，SLAVE_RANK 是复制偏移量的差值，即与主节 点的数据差距越小的从节点，等待的时间就会越小，越有可能成为新的主节点。

##### 转移过程

新选举出的主节点会首先会执行SLAVEOF no one，停止对旧主节点的复制动作。然后会将旧主节点的槽位指派给自己。最后向集群发送广播消息，让集群中的其他节点知道新的主节点已经接管所有的槽位，旧主节点已经完成下线动作。这样就完成了整个的故障转移过程，新的主节点将开始接受与自己负责的槽位有关的数据。

---

### MongoDB VS Mysql

字节

1、MongoDB和MySQL的区别?MongoDB为什么读写快?

#### 题解

MySQL与MongoDB都是开源的常用数据库，但是MySQL是传统的关系型 数据库，MongoDB则是非关系型数据库，也叫文档型数据库，是一种NoSQL的 数据库。它们各有各的优点，关键是看用在什么地方。所以我们所熟知的那些 SQL语句就不适用于MongoDB了，因为SQL语句是关系型数据库的标准语言。

#### MySQL特点

1. 在不同的引擎上有不同的存储方式。
2. 查询语句是使用传统的sql语句，拥有较为成熟的体系，成熟度很高。 
3. 开源数据库的份额在不断增加，mysql的份额也在持续增长。
4. 缺点就是在海量数据处理的时候效率会显著变慢。

#### MongoDB

非关系型数据库(nosql ),属于文档型数据库。先解释一下文档的数据 库，即可以存放xml、json、bson类型系那个的数据。这些数据具备自述性， 呈现分层的树状数据结构。数据结构由键值(key=>value)对组成。

1. 存储方式:虚拟内存+持久化。
2. 查询语句:是独特的MongoDB的查询方式。
3. 适合场景:事件的记录，内容管理或者博客平台等等。
4. 架构特点:可以通过副本集，以及分片来实现高可用。
5. 数据处理:数据是存储在硬盘上的，只不过需要经常读取的数据会被加载到内存中，将数据存储在物理内存中，从而达到高速读写。

##### mongodb会比mysql快

1. 首先是内存映射机制，数据不是持久化到存储设备中的，而是暂时存储 在内存中，这就提高了在IO上效率以及操作系统对存储介质之间的性能 损耗。(毕竟内存读取最快)
2. 其次，NoSQL并不是不使用sql，只是不使用关系。没有关系的存在， 就比如每个数据都好比是拥有一个单独的存储空间，然后一个聚集索引 来指向。搜索性能一定会提高的。
3. mongodb 天生支持分布式部署，如果数据量足够大可以使用 mongodb的分片集群，这样也是mongodb比较快的一个原因

##### 使用场景区别

MongoDB 是一种文档型数据库，由于它不限制数据量和数据类型， 它是高容量环境下最合适的解决方案。由于MongoDB具备云服务需要的水平 可伸缩性和灵活性，它非常适合云计算服务的开发。另外，它降低了负载，简 化了业务或项目内部的扩展，实现了高可用和数据的快速恢复。

尽管 MongoDB 有那么多优点，但 MySQL 也在某些方面优于 MongoDB，例如可靠性和数据一致性。另外如果优先考虑安全性，MySQL就是安全性最高的 DBMS 之一。

而且，当应用程序需要把多个操作视为一个事务(比如会计或银行系 统)时，关系数据库是最合适的选择。除了安全性，MySQL的事务率也很高。 实际上，MongoDB 支持快速插入数据，而MySQL相反，它支持事务操作，并关注事务安全性。

总体上看，如果项目的数据模式是固定的，而且不需要频繁变更，推荐使用MySQL，因此项目维护容易，而且确保了数据的完整性和可靠性。

另一方面，如果项目中的数据持续增加，而且数据模式不固定， MongoDB 是最合适的选择。由于它属于非关系数据库，数据可以自由使用， 不需要定义统一的数据结构，所以对数据进行更新和查询也很方便。MongoDB 通常用于需要对内容进行管理、处理物联网相关业务、进行实时分析等功能的 项目中。

---

### ES

阿里云

1、什么是es的倒排索引?

#### 题解

想理解倒排索引，首先先思考一个问题，获取某个文件夹下所有文件名中包含Spring的文件

- 确定要搜索的文件夹
- 遍历文件夹下所有文件
- 遍历文件夹下所有文件

这种思维可以理解为是一种正向思维的方式，从外往内，根据key找value。这种方式可以理解为正向索引。

而ElasticSearch为了提升查询效率，采用反向思维方式，根据value找key。

![](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202305221526524.png)
