---
layout: post
title: "redis底层数据结构"
date: 2022-09-06 16:02:30
categories: NoSQL
keywords: "redis底层数据结构"
description: "redis底层数据结构介绍"
---

## 一、前置知识&总览

### 对象结构全貌

Redis的每种对象其实都由**对象结构(redisObject)** 与 对应编码的数据结构组合而成，每种对象类型对应若干种编码方式，不同的编码方式所对应不同的底层数据结构

```C++
/*
 * Redis 对象
 */
typedef struct redisObject {

    unsigned type:4;     // 类型

    unsigned encoding:4; // 编码方式

    unsigned lru:24;     // 记录最后一次访问时间（相对于lru_clock）; 或者LFU（最少使用的数据：8位频率，16位访问时间）

    int refcount;        // 引用计数

    void *ptr;           // 指向底层数据结构实例

} robj;
```

对应如下结构

![redisObject](/img/redis/redisObject.png)

- type记录了对象所保存的值的类型

```c++
/*
* 对象类型
*/
#define OBJ_STRING 0 // 字符串
#define OBJ_LIST 1   // 列表
#define OBJ_SET 2    // 集合
#define OBJ_ZSET 3   // 有序集
#define OBJ_HASH 4   // 哈希表
```

- encoding记录了对象所保存的值的编码

```c++
/*
* 对象编码
*/
#define OBJ_ENCODING_RAW 0       /* Raw representation */
#define OBJ_ENCODING_INT 1       /* Encoded as integer */
#define OBJ_ENCODING_HT 2        /* Encoded as hash table */
#define OBJ_ENCODING_ZIPLIST 5   /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6    /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8    /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10   /* Encoded as a radix tree of listpacks */
```

- ptr是一个指针，指向实际保存值的数据结构
- lru属性记录了对象最后一次被命令程序访问的时间

<div style="display:flex;justify-content: center; align-items:center; width: 100%;text-align:center;line-height:40px ">
    <div style="font-weight:bold">基本数据类型对应的底层数据结构</div>
</div>

| 基本数据类型 | 底层数据结构                                  |
| ------------ | --------------------------------------------- |
| String       | 简单动态字符串（SDS - simple dynamic string） |
| Hash         | 压缩列表（ZipList），哈希表（Dict）           |
| List         | 双向链表（QuickList），压缩列表（ZipList）    |
| Set          | 整数数组（IntSet），哈希表（Dict）            |
| ZSet         | 压缩列表（ZipList）， 跳表（ZSkipList）       |

<div style="display:flex;justify-content: center; align-items:center; width: 100%;text-align:center;line-height:40px ">
    <div style="font-weight:bold">Redis所有编码方式以及底层数据结构之间的关系</div>
</div>
<table style="background:white">
	<thead>
		<tr>
			<th>数据类型</th>
			<th>编码类型</th>
			<th>底层数据结构</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td rowspan="3">String</td>
			<td>OBJ_ENCODING_RAW</td>
			<td rowspan="2">SimpleDynamicString</td>
		</tr>
		<tr>
			<td>OBJ_ENCODING_EMBSTR</td>
		</tr>
    <tr>
			<td>OBJ_ENCODING_INT</td>
      <td>直接存储在redisObject的Ptr</td>
		</tr>
		<tr>
			<td rowspan="2">List</td>
			<td>OBJ_ENCODING_QUICKLIST</td>
			<td>QuickList</td>
		</tr>
    <tr>
			<td>OBJ_ENCODING_ZIPLIST</td>
			<td>ZipList</td>
		</tr>
		<tr>
			<td rowspan="2">Hash</td>
			<td>OBJ_ENCODING_ZIPLIST</td>
			<td>ZipList</td>
		</tr>
		<tr>
			<td>OBJ_ENCODING_HT</td>
			<td>Dict</td>
		</tr>
		<tr>
			<td rowspan="2">Set</td>
			<td>OBJ_ENCODING_INTSET</td>
			<td>IntSet</td>
		</tr>
		<tr>
			<td>OBJ_ENCODING_HT</td>
			<td>Dict</td>
		</tr>
		<tr>
			<td rowspan="2">ZSet</td>
			<td>OBJ_ENCODING_ZIPLIST</td>
			<td>ZipList</td>
		</tr>
		<tr>
			<td>OBJ_ENCODING_SKIPLIST</td>
			<td>ZSkipList</td>
		</tr>
	</tbody>
</table>


### 命令的类型检查和多态行为

当执行一个处理数据类型命令的时候，redis执行以下步骤

1. 根据给定的key，在数据库字典中查找和他相对应的redisObject，如果没找到，就返回NULL
2. 检查redisObject的type属性和执行命令所需的类型是否相符，如果不相符，返回类型错误
3. 根据redisObject的encoding属性所指定的编码，选择合适的操作函数来处理底层的数据结构
4. 返回数据结构的操作结果作为命令的返回值

> OBJECT ENCODING [key] 命令可以查看底层对象编码

## 二、SimpleDynamicString

### 结构介绍

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;    
    uint8_t alloc; 
    unsigned char flags;
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; 
    uint16_t alloc; 
    unsigned char flags; 
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; 
    uint32_t alloc; 
    unsigned char flags;
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; 
    uint64_t alloc; 
    unsigned char flags; 
    char buf[];
};
```

- `len` 保存字符串的长度

- `buf[]` 数组用来保存字符串的每个元素

- `alloc`分别以8位, 16位, 32位, 64位表示整个SDS，不包含header（sds是指向buf的char*，所以结构体中buf都是放在最后。header就是len、alloc、flags）和结束字符\0

- `flags` 以低三位标示着用着那种类型的SDS

  ```c
  #define SDS_TYPE_8  1
  #define SDS_TYPE_16 2
  #define SDS_TYPE_32 3
  #define SDS_TYPE_64 4
  #define SDS_TYPE_MASK 7
  #define SDS_TYPE_BITS 3
  ```

<img src="assets/sds.png" alt="SDS 结构" style="zoom:67%;" />

### 分配细节

**空间预分配**

空间预分配是用于优化 SDS 字符串增长操作的，简单来说就是当字节数组空间不足触发重分配的时候，总是会预留一部分空闲空间。这样的话，就能减少连续执行字符串增长操作时的内存重分配次数。

有两种预分配的策略：

- len 小于 1MB 时：每次重分配时会多分配同样大小的空闲空间；
- len 大于等于 1MB 时：每次重分配时会多分配 1MB 大小的空闲空间。

---

**惰性空间释放**

惰性空间释放是用于优化 SDS 字符串缩短操作的。简单来说就是当字符串缩短时，并不立即使用内存重分配来回收多出来的字节，而是用 free 属性记录，等待将来使用。SDS 也提供直接释放未使用空间的 API，在需要的时候，也能真正的释放掉多余的空间。

### 实现细节技巧与优点

**实现的技巧**

SDS结构体取消了字节对齐，然后又将指针直接指向了结构体的尾部的`buf`对象。指针指向的位置完全兼容了c的字符串操作，如果需要得到其它信息则使用了`flags = s[-1]`的下标访问操作得到类型之后再做计算。

**优点**

- 常数复杂度获取字符串长度

  由于 len 属性的存在，我们获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。而对于 C 语言，获取字符串的长度通常是经过遍历计数来实现的，时间复杂度为 O(n)

- 杜绝缓冲区溢出

  C语言中使用 `strcat`  函数来进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而对于 SDS 数据类型，在进行字符修改的时候，会首先根据记录的len属性检查内存空间是否满足需求，如果不满足，会进行相应的空间扩展，然后在进行修改操作，所以不会出现缓冲区溢出

- 减少修改字符串的内存重新分配次数

  C语言由于不记录字符串的长度，所以如果要修改字符串，必须要重新分配内存。而对于SDS，由于`len`属性和`alloc`属性的存在，对于修改字符串SDS实现了`空间预分配`和`惰性空间释放`两种策略

- 二进制安全

  因为C字符串以空字符作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此C字符串无法正确存取。而所有SDS的API都是以处理二进制的方式来处理 `buf` 里面的元素，并且 SDS 不是以空字符串来判断是否结束，而是以len属性表示的长度来判断字符串是否结束

- 兼容C字符串函数

## 三、IntSet

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

- `encoding`表示编码方式，取值有三个：INTSET_ENC_INT16, INTSET_ENC_INT32, INTSET_ENC_INT64

- `length`代表其中存储的整数的个数

- `contents`实际存储数值数组，数组元素从小到大有序排序，且数组中不包含任何重复项。

> 虽然IntSet结构将contents属性声明为int8_t类型的数组，但实际上contents数组并不保存任何int8_t类型的值，contents数组的真正类型取决于encoding属性的值

**整数集合的升级**

当在一个int16类型的整数集合中插入一个int32类型的值，整个集合的所有元素都会转换成32类型。 整个过程有三步：

- 根据新元素的类型（比如int32），扩展整数集合底层数组的空间大小，并为新元素分配空间
- 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变
- 最后改变encoding的值，length+1

集合不支持降级

## 四、Dict

```c
/*
 * 哈希表
 */
typedef struct dictht {
    dictEntry **table;       // 哈希表数组
    unsigned long size;      // 哈希表大小
    unsigned long sizemask;  // 哈希表大小掩码，用于计算索引值
    unsigned long used;      // 哈希表数组的长度
} dictht;

/*
 * 哈希表节点
 */
typedef struct dictEntry {
    void *key;               // 键
    union {                  // 值
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;  // 链的后继节点
} dictEntry;

/*
 * 字典
 * 每个字典使用两个哈希表，用于实现渐进式 rehash
 */
typedef struct dict {
    dictType *type;          // 特定于类型的处理函数
    void *privdata;          // 类型处理函数的私有数据
    dictht ht[2];            // 哈希表（2 个）
    long rehashidx;          // 记录rehash进度的标志，值为-1表示rehash未进行
    int16_t pauserehash;     // 如果大于0则代表rehash暂停
} dict;
```

![Dict](/img/redis/Dict.png)

**扩容和收缩**

当哈希表保存的键值对太多或者太少时，就要通过rerehash(重新散列）来对哈希表进行相应的扩展或者收缩。

- 如果执行扩展操作，会基于原哈希表创建一个大小等于 ht[0].used*2n 的哈希表（也就是每次扩展都是根据原哈希表已使用的空间扩大一倍创建另一个哈希表）。相反如果执行的是收缩操作，每次收缩是根据已使用空间缩小一倍创建一个新的哈希表
- 重新利用上面的哈希算法，计算索引值，然后将键值对放到新的哈希表位置上
- 所有键值对都迁徙完毕后，释放原哈希表的内存空间

**渐近式 rehash**

渐进式rehash是指扩容和收缩操作不是一次性、集中式完成的，而是分多次、渐进式完成的。如果保存在Redis中的键值对只有几个几十个，那么 rehash 操作可以瞬间完成，但是如果键值对有几百万，几千万甚至几亿，那么要一次性的进行rehash，势必会造成Redis一段时间内不能进行别的操作。所以Redis采用渐进式rehash，<span style="color:blue">在进行渐进式rehash期间，字典的删除查找更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找。但是进行增加操作，一定是在新的哈希表上进行的。</span>

## 五、ZSkipList

跳跃表结构在Redis中的运用场景只有一个，那就是作为有序列表ZSet的使用。跳跃表的性能可以保证在查找，删除，添加等操作的时候在对数期望时间内完成，这个性能是可以和平衡树来相比较的，而且在实现方面比平衡树要优雅，这就是跳跃表的长处。跳跃表的缺点就是需要的存储空间比较大，属于利用空间来换取时间的数据结构。

```c
typedef struct zskiplistNode {
    sds ele;                              // 数据
    double score;                         // 分数
    struct zskiplistNode *backward;       // 后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;    // 前进指针
        unsigned int span;                // 跨度
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;   // 表头节点和表尾节点
    unsigned long length;                  // 表中节点的数量
    int level;                             // 表中层数最大的节点的层数
} zskiplist
```

![ZSkipList](/img/redis/ZSkipList.png)

zskiplist结构

- `header`/`tail`指向跳跃表的表头/尾节点
- `level`记录目前跳跃表内，层数最大的那个节点层数（表头节点的层数不计算在内，头节点一直32个level）
- `length`记录跳跃表的长度，也就是跳跃表目前包含节点的数量（表头节点不计算在内，即真实数据量）

zskiplistNode结构

- `ele`存储的元素
- `score`元素分数，从小到大排列

- `backward`指向结点的前一个紧邻结点

- `level`字段，用以记录所有结点（除过头节点外）每个结点中最多持有32个zskiplistLevel结构. 实际数量在结点创建时，按幂次定律随机生成，每加一层概率为1/4
  - `forward`字段指向比自己得分高的下个节点
  - `span`字段代表forward字段指向的结点, 距离当前结点的距离. 紧邻的两个结点之间的距离定义为1

## 六、ZipList

ZipList是一个经过特殊编码达到双向链表效果的结构，它的设计目标就是为了提高存储和访问效率。它能以O(1)的时间复杂度在表的两端提供push和pop操作。又因为ziplist是一个内存连续的集合，所以ziplist遍历只要通过当前节点的指针加上当前节点的长度或减去上一节点的长度 ，即可得到下一个节点的数据或上一个节点的数据，这样就省去的指针从而节省了存储空间，又因为内存连续所以在数据读取上的效率也远高于普通的链表

![zipList](/img/redis/zipList.png)

ZipList结构

- `zlbytes`存储的是整个ziplist所占用的内存的字节数（包含他自己）
- `zltail`指的是ziplist中最后一个entry的偏移量. 用于快速定位最后一个entry, 以快速完成pop等操作
- `zllen` 记录entry的数量（Redis作了特殊的处理：当实体数超过2^16，该值被固定为2^16-1所以这种时候要知道所有实体的数量就必须要遍历整个结构了，效率会降低）
- `entry`真正存数据的结构。
- `zlend`固定为 255 (0xFF)，作为ziplist的结束标识

Entry结构

- `prevlen`前一个entry的大小
  - 如果前一个元素长度小于254（255用于zlend）的时候，prevlen长度为1个字节，值即为前一个entry的长度
  - 如果前一个元素长度大于等于254的时候，prevlen用5个字节表示，第一字节设置为254，后面4个字节存储一个无符号整型，表示前一个entry的长度，例如长度300则表示为 0xFE 00 00 00 12C 

- `encoding`不同的情况下值不同，用于表示当前entry的类型和长度，如果存储时数字则可能不需要使用entry-data存储信息，而是直接受用encoding存储数据
- `entry-data`存储entry表示的数据

ZipList缺点

Ziplist不适合保存很多的元素，因为插入、删除、更新操作需要重新分配地址并且复制原来的元素，所以在保存数据量较少的数据时，Ziplist才有优势。而且可能设计的结构会引发**连锁更新**问题，<span style="color:blue">*ZipList在v7.0被`ListPack`替代*</span>

**连锁更新**

假设有这样的一个ziplist，每个节点都是等于253字节的。新增了一个大于等于254字节的新节点，由于之前的节点prevlen长度是1个字节。为了要记录新增节点的长度所以需要对节点1进行扩展，由于节点1本身就是253字节，再加上扩展为5字节的pervlen则长度超过了 254字节，这时候下一个节点又要进行扩展了。噩梦就开始了。
![连锁更新](/img/redis/连锁更新.png)

## 七、QuickList

QuickList实际上就是zipList和linkedList的混合体，它将linkedList按段切分，每一段使用zipList来紧凑存储，多个zipList之间使用双向指针串接起来。避免了过多的内存碎片。

![QuickList](/img/redis/QuickList.png)

QuickList结构

- `fill`的值影响着每个链表结点中, ziplist的长度

  - -1 不超过 4kb

  - -2 不超过 8kb

  - -3 不超过 16kb

  - -4 不超过 32kb

  - -5 不超过 64kb

  - 当数值为正数时, 代表以entry数目限制单个ziplist的长度

- `count`所有ziplist中entry数量

- `len`所有quicklistNodes节点数量

- `compress`的值影响着zipList指针指向的是原生的ziplist，还是经过压缩包装后的

- `head`和`tail`代表着头尾指针

QuickListNode结构

- `prev`和`next`代表前/后一节点指针
- `ziplist`是指向zipList的指针
- `sz`统计指向的ziplist实际占用内存大小。（如果ziplist被压缩了，那么这个sz的值仍然是压缩前的ziplist大小）
- `count`统计ziplist里面包含的数据项个数
- `encoding `表示是否压缩
- `container `指存储类型，目前使用固定值2表示使用ziplist存储
- `recompress`现在数据是否被解压了
- `attempted_compress`节点不能被压缩因为太小了
- `extra `占位没用上





## 八、基本数据类型底层编码场景

### String

**OBJ_ENCODING_INT**

当字符串键值的内容可以用一个 **64位有符号整形** 来表示时，Redis会将键值转化为long型来进行存储，此时即对应 `OBJ_ENCODING_INT` 编码类型，而且 Redis 启动时会预先建立**10000**个分别存储**0~9999**的 redisObject变量作为共享对象，这就意味着如果set字符串的键值在0~10000之间的话，则可以**直接指向共享对象**而不需要再建立新对象，此时键值不占空间

<img src="/img/redis/str_int.png" alt="str_int" style="zoom:50%;" />

**OBJ_ENCODING_EMBSTR**

对于长度小于44的字符串，Redis键值采用`OBJ_ENCODING_EMBSTR`方式，表示嵌入式的String。从内存结构上来讲即字符串SDS结构体与其对应的redisObject对象分配在**同一块连续的内存空间**，这就仿佛字符串SDS嵌入在redisObject对象之中一样

<img src="/img/redis/str_emb.png" alt="str_emb" style="zoom:50%;" />

**OBJ_ENCODING_RAW**

当字符串长度大于**44**的时，Redis则会将键值的内部编码方式改为`OBJ_ENCODING_RAW` 格式，这与上面的`OBJ_ENCODING_EMBSTR`编码方式的不同之处在于此时动态字符串SDS的内存与其依赖的redisObject的**内存不再连续**了，且当在原有字符串上进行追加时创建的新字符串的编码时RAW

<img src="/img/redis/str_raw.png" alt="str_raw" style="zoom:50%;" />

### List

满足如下条件List会用ZipList作为实现，否则使用QuickList

（1）列表对象保存的所有字符串元素的长度都小于64字节

```config
list-max-ziplist-value 64  #保存的所有字符串元素的长度都小于64字节。
```

（2）列表对象保存的元素数量小于512个

```config
list-max-ziplist-entries 512 #保存的元素数量小于512个。
```

### Hash

Hash的底层存储可以使用Ziplist和Dict。当Hash对象可以同时满足一下两个条件时，哈希对象使用Ziplist编码（一个元素存key，下一个元素存value）

（1）哈希对象保存的所有键值对的键和值的字符串长度都小于64字节

```config
hash-max-ziplist-value 64
```

（2）哈希对象保存的键值对数量小于512个

```config
hash-max-ziplist-entries 512
```

### Set

元素都是整数类型，就用IntSet存储，否则使用Dict存储

### ZSet

发送如下事情，ZSet底层实现会从ZipList变为ZSkipList

（1）当sorted set中的元素个数，即(数据, score)对的数目超过128的时

```config
zset-max-ziplist-entries 128
```

（2）当sorted set中插入的任意一个数据的长度超过了64的时

```config
zset-max-ziplist-value 64
```
