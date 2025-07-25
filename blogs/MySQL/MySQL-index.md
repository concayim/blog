---
title: MySQL-索引
tags: 
  - MySQL
date: 2023-04-03 15:27:21
categories:
  - MySQL
---

## MySQL索引

### 1.索引数据结构

哈希表，完全平衡二叉树，B树，B+树等

### 2. 为什么索引的数据结构选择B+树？

#### A） 哈希

可以快速精确查询，但是在进行范围查询的时候不方便

#### B） 完全平衡二叉树

插入数据慢，查询也慢，能支持范围查询，

#### C） B树

同样的元素，B树的表示要比完全平衡二叉树要矮，原因在于B树的一个节点可以存储多个元素

度：节点能够插入的最大的值

#### D） B+树

把所有的非叶子节点在叶子节点上冗余了一份（为什么要进行冗余？）

叶子节点之间用箭头连接起来，有什么好处？

底层的叶子节点是一个递增的状态，每当查到一个元素之后，可以把后面的元素遍历出来，方便范围查找

总结：

按照索引去查询，读取的是磁盘中的文件，涉及到了磁盘的IO。树的高度越低，io效率越高。（所以排除完全平衡二叉树）但是建立索引之后插入数据很慢

### 3. B+树的一个节点存多少个元素合适？（一个节点多大合适？）

引入：固态读写为什么比机械快？

固态使用的是电路进行读写，而机械硬盘使用的是机械运动

机械硬盘：

操作系统告诉一个地址，机械硬盘中有一个定位驱动器，把地址翻译成磁盘能够识别的地址，定位到数据的位置，然后用取数臂，控制磁臂转到对应的磁道

计算机局部性原理：

当一个数据被用到时，其附近的数据也通常马上被使用

就是说，当你的操作系统向磁盘发送请求，要取1kb的时候，磁盘并不认为你要取出1kb的数据，会多返一点数据给你，会返回一页，一页的大小是4kb

解答：

一个节点多大合适。就是16k（一页）

SHOW GLOBAL STATUS LIKE 'innodb_page_size'（innodb的页大小）

显示出来的结果是16384，就是16k

Mysql中innodb和myisam使用B+树（为什么不使用B树？）

B+树的非叶子节点不存储数据，B树的非叶子和叶子节点都存储数据。所以导致非叶子节点存储的索引值会更少，树的高度会比B+树高，平均的IO会比较低，加上B+树的叶子节点之间会有指针相连，也方便进行范围查询。

如果，将非叶子节点也存入data，相对来说，索引的key值就会减少，树就会变高，效率就会变慢，不存的原因是为了提高整个索引的查找

效率，而不仅仅只是一个节点

### 4. MyIsam

#### A.主键索引

Data存的是地址

#### B.辅助索引

innodb

##### a.主键索引（如果没有存主键索引，就去找唯一索引，如果唯一索引都没有，就会默认自动建立一个主键索引）

data中存的就是真实的数据（比myisam少一次io）

##### b.辅助索引（对普通列建立一个索引）

data存的是主键，多几次磁盘io

总结：

非叶子节点的主键假设是bigint类型，那么长度是8b，指针大小是6b。那么一页里就可以存16k/14=1170个（主键+指针），那么一颗高度为2的B+树能存储的（主键+指针）为18720条

### 5. 索引的分类

普通索引index：加速查询

唯一索引

联合索引

全文索引

### 6. 索引优点

可以通过建立唯一索引或者主键索引，保证数据表中每一行数据的唯一性建立索引可以大大提高检索的数据，以及减少表的检索行数在表的连接条件，可以加速表与表直接的相连，建立索引，在查询中使用索引可以提高性能

### 7. 分析索引使用情况

Explain里面的type，可以查看是否有使用到索引

### 8. 哪些情况会造成索引失效

如果条件中有or，索引字段的值不能有null值。对于多列索引，不是使用的第一部分，则不会使用索引，Like以%开头，如果列类型是字符串，那一定要加单引号

使用表达式或者函数会使索引失效
