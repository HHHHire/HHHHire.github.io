---
layout:     post
title:      redis 数据结构
subtitle:   redis 数据结构
date:       2021-4-18
author:     HUA
header-img: img/home-bg-o.jpg
catalog: 	 true
tags:
    - 数据库
    - redis
---
# redis 数据结构

## 数据类型

string、hash、list、set、zset

Redis内部使用一个redisObject对象来表示所有的key和value。redisObject主要的信息包括数据类型（type）、编码方式(encoding)等。type代表一个value对象具体是何种数据类型，encoding是不同数据类型在redis内部的结构

![](https://raw.githubusercontent.com/HHHHire/image/master/img/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%841.png)



## 内存结构

### sds 简单动态字符串

![](https://raw.githubusercontent.com/HHHHire/image/master/img/20210414204803.png)

* len：字符串的长度，buff数组中已使用的字节数
* alloc：分配的buf数组长度，不包括头和空字符结尾 
* buff：字节数据，用于保存字符串

**相较于C字符串**

* 获取字符串长度的复杂度为O(1)
* 可以保存文本、二进制数据

**空间预分配**

* 对SDS修改后，len的长度小于 1M，那么程序将分配和 len 相同长度的未使用空间
* 对SDS修改后，len的长度大于1M，那么程序将分配1M的未使用空间

### dict 字典

![](https://raw.githubusercontent.com/HHHHire/image/master/img/20210414201521.png)

* dictht：两个hash表，ht[0] ht[1] 只有在重哈希过程过 ht[0] 和 ht[1] 才都有效。里面的 dictEntry 就是具体存放数据的
* rehashidx：-1 时表示此时没有重哈希，否则，就是正在重哈希过程中
* dictEntry：如果hash值重复，就会形成链表
* size：dictEntry数组的长度，2的指数倍
* sizemask：hash表大小的掩码表示，和key的hash值一起决定一个键放在哪个位置
* used：记录数据个数，他和size的比值越大，hash冲突概率越高

**增量式重哈希（渐进式）**：当hash表的数据个数达到一定值时，需要扩容，相较于直接重哈希，增量式更加不会影响性能。他是每次只重哈希一部分数据，每当有数据进行CRUD时，同时对该数据进行重哈希。在所有键值对都迁移到了ht[1]后，释放ht[0]，将ht[1]设置为ht[0]，再新建个空哈希表ht[1]。

**具体过程如下：**

1. 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2. 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值加一。
4. 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1]， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。
5. 过程中的查询都先到 ht[0] 再去 ht[1]，新增都去 ht[1]

### ziplist 压缩列表

为了节约内存开发，由连续的内存块组成，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值

![](https://raw.githubusercontent.com/HHHHire/image/master/img/20210414110848.png)

entry:

![](https://raw.githubusercontent.com/HHHHire/image/master/img/20210414110904.png)

* previous_entry_length：记录前一个节点的长度，如果前一个节点的长度小于254，那么previous_entry_length本身的大小就是1字节，否则为5字节

### skiplist 跳跃表

![](https://raw.githubusercontent.com/HHHHire/image/master/img/20210414110925.png)

跳跃表

```c
#define ZSKIPLIST_MAXLEVEL 32
#define ZSKIPLIST_P 0.25
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

跳跃表节点

```c
typedef struct zskiplistNode {
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

- 开头定义了两个常量，ZSKIPLIST_MAXLEVEL和ZSKIPLIST_P，一个是MaxLevel 最大层数，一个是p随机的概率。
- zskiplist定义了真正的skiplist结构，它包含：
  - 头指针header和尾指针tail。
  - 链表长度length，即链表包含的节点总数。注意，新创建的skiplist包含一个空的头指针，这个头指针不包含在length计数中。
  - level表示skiplist的总层数，即所有节点层数的最大值
- zskiplistNode定义了skiplist的节点结构。
  - obj字段存放的是节点数据，它的类型是一个string robj。本来一个string robj可能存放的不是sds，而是long型，但zadd命令在将数据插入到skiplist里面之前先进行了解码，所以这里的obj字段里存储的一定是一个sds。这样做的目的应该是为了方便在查找的时候对数据进行字典序的比较，而且，skiplist里的数据部分是数字的可能性也比较小。
  - score字段是数据对应的分数。
  - backward字段是指向链表前一个节点的指针（前向指针）。节点只有1个前向指针，所以只有第1层链表是一个双向链表。
  - level[]存放指向各层链表后一个节点的指针（后向指针）。每层对应1个后向指针，用forward字段表示。另外，每个后向指针还对应了一个span值，它表示当前的指针跨越了多少个节点。span用于计算元素排名(rank)，这正是前面我们提到的Redis对于skiplist所做的一个扩展。

跳跃表的CRUD是从上到下的进行查找的，而每个节点的层数则是随机的，每当插入一个新的节点，他都是随机生成一个随机的层数（redis自己的算法）。

## 编码转换

每一种数据类型，是支持多种数据结构的，不同情况下使用不同的数据结构，这就通过编码转换来实现。

### String

* int：当string对象全是数字，就会使用int编码，实际上是long类型
* embstr：字符串或浮点数长度小于等于39字节，就会使用embstr编码方式来存储，embstr存储内存一般很小，所以redis一次性分配且内存连续(效率高)。 是一种特殊的sds
* raw：当一个字符串或浮点数长度大于39字节，就使用SDS来保存，编码为raw，由于不确定值的字节大小，所以键和值各分配各的，所以就分配两次内存(回收也是两次)，同理它一定不是内存连续的。

### List

* ziplist -> linkedlist：列表对象的所有字符串元素的长度大于等于64字节 & 列表元素数大于等于512. 反之，小于64和小于512会使用ziplist而不是用linkedlist。

### Hash

* ziplist -> hashtable： 哈希对象所有键和值字符串长度大于等于64字节 & 键值对数量大于等于512，修改选项：`list-max-ziplist-value`和`list-max-ziplist-entriess` 

### Set

* intset：集合对象所有元素都是整数，集合对象元素数不超过512个
* intset -> hashtable ：元素不都是整数 || 元素数大于等于512 

### Zset

* ziplist：列表节点中一个成员，一个分值，从小到大排序
* skiplist & dict：其实只用skiplist也能实现，但是查找效率没有dict高。成员、分值都存在于两种数据结构，他们通过指针共享，不会造成内存的浪费。
* ziplist -> skiplist & dict：有序集合元素数 >= 128 & 含有元素的长度 >= 64 

## GEO

geo 就是用到了 zset 的特性，将经纬度经过计算转换成分值存入zset中。

![](https://raw.githubusercontent.com/HHHHire/image/master/img/20210415004433.png)

```shell
############### GEO ###################
# 添加位置信息 longitude:经度 latitude:纬度 member:成员  ，会覆盖旧数据
geoadd key longitude latitude member
# 获取成员的经纬度
geopos key member
# 查看成员间距离 unit: m,km,mi(英里),ft(尺)
geodist key member1 member2 [unit]
# 获取指定范围内的地理信息位置集合
# 以指定的经纬度为中心计算，radius:半径,withcoord:结果中包含经纬度,withdist:结果中包含距离,withhash:结果中包含geohash,COUNT count:指定数量，store key:将返回结果的地理位置信息保存到指定键,storedist key:将返回结果里中心点的距离保存到指定键
georadius key longitude latitude radius m|km|ft|mi [withcoord] [withdist]
[withhash] [COUNT count] [asc|desc] [store key] [storedist key]
# 以成员为中心计算
georadiusbymember key member radius m|km|ft|mi [withcoord] [withdist]
[withhash] [COUNT count] [asc|desc] [store key] [storedist key]
# 获取geohash
geohash key member
```
