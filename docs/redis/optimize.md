# redis的内存优化

Redis所有的数据都在内存中，而内存又是非常宝贵的资源。对于如何优化内存使用一直是非常重要的。因此，redis的内存优化是非常重要的。

## 一. redisObject对象
Redis存储的所有值对象在内部定义为redisObject结构体，内部结构如下所示。
```c
#define LRU_BITS 24
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */
#define LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */

typedef struct redisObject {
    // 对象的数据类型，占4bits，共5种类型
    unsigned type:4;        
    // 对象的编码类型，占4bits，共10种类型
    unsigned encoding:4;

    //least recently used
    //实用LRU算法计算相对server.lruclock的LRU时间
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */

    //引用计数
    int refcount;

    //指向底层数据实现的指针
    void *ptr;
} robj;
```

Redis存储的数据都使用redisObject来封装，包括`string`, `hash`, `list`, `set`, `zset`在内的所有数据类型。  
下面针对每个字段做详细说明：

### 1.type字段:
表示当前对象使用的数据类型，Redis主要支持5种数据类型: `string`, `hash`, `list`, `set`, `zset`。  
可以使用` type key `命令查看对象所属类型，`type`命令返回的是值对象类型，键都是`string`类型。

### 2.encoding字段:
表示Redis内部编码类型，encoding在Redis内部使用，代表当前对象内部采用哪种数据结构实现。  
Redis内部编码方式对于优化内存非常重要 ，同一个对象采用不同的编码实现内存占用存在明显差异。

### 3.lru字段:
记录对象最后一次被访问的时间。  
当配置了`maxmemory`和`maxmemory-policy=volatile-lru | allkeys-lru `时，用于辅助LRU算法删除键数据。  
可以使用`object idletime key`命令在不更新lru字段情况下查看当前键的空闲时间。

> 可以使用`scan`迭代出所有的key，配合`object idletime`命令批量查询哪些键长时间未被访问，找出长时间不访问的键进行清理降低内存占用。

### 4.refcount字段:
记录当前对象被引用的次数，用于通过引用次数回收内存，当refcount=0时，可以安全回收当前对象空间。  
使用`object refcount key`获取当前对象引用。当对象为整数且范围在`[0-9999]`时，Redis可以使用共享对象的方式来节省内存。

### 5. *ptr字段:
与对象的数据内容相关，如果是整数直接存储数据，否则表示指向数据的指针。
Redis在3.0之后对值对象是字符串且长度<=39字节的数据，内部编码为embstr类型，字符串sds和redisObject一起分配，从而只要一次内存操作。

> 高并发写入场景中，在条件允许的情况下建议字符串长度控制在39字节以内，减少创建redisObject内存分配次数从而提高性能。

## 二. 缩减键值对象
降低Redis内存使用最直接的方式就是缩减键（key）和值（value）的长度。

- key长度：在完整描述业务情况下，键值越短越好。
- value长度：值对象缩减比较复杂，常见需求是把业务对象序列化成二进制数组放入Redis。首先应该在业务上精简对象，去掉不必要的属性。其次在序列化工具选择上，使用更高效的序列化工具。

值对象除了存储二进制数据之外，通常还会使用通用格式存储数据比如:json，xml等作为字符串存储在Redis中。  
在内存紧张的情况下，可以使用通用压缩算法压缩json,xml后再存入Redis，从而降低内存占用，例如使用GZIP压缩后的json可降低约60%的空间。

> 当频繁压缩解压json等文本数据时，开发人员需要考虑压缩速度和计算开销成本，这里推荐使用google的Snappy压缩工具，在特定的压缩率情况下效率远远高于GZIP等传统压缩工具，且支持所有主流语言环境。

## 三. 共享对象池

对象共享池指Redis内部维护`[0-9999]`的整数对象池。  
创建大量的整数类型redisObject存在内存开销，每个redisObject内部结构至少占16字节，甚至超过了整数自身空间消耗。所以Redis内存维护一个`[0-9999]`的整数对象池，用于节约内存。  
除了整数值对象，其他类型如`list`, `hash`, `set`, `zset`内部元素也可以使用整数对象池。因此在满足需求的前提下，尽量使用整数对象以节省内存。
整数对象池在Redis中通过变量REDIS_SHARED_INTEGERS定义，不能通过配置修改。可以通过object refcount 命令查看对象引用数验证是否启用整数对象池技术。

使用整数对象池究竟能降低多少内存？让我们通过测试来对比对象池的内存优化效果，如下表所示。

操作说明      | 是否对象共享	 | key大小	     | value大小	    | used_mem   | used_memory_rss
------------ | ------------- | ------------- | ------------ | ---------- | ----------------
插入200万     |      否       |     20字节     | [0-9999]整数 |  199.91MB	 |  205.28MB
插入200万     |      是       |     20字节     | [0-9999]整数 |  138.87MB	 |  143.28MB

使用共享对象池后，相同的数据内存使用降低30%以上。  
可见当数据大量使用[0-9999]的整数时，共享对象池可以节约大量内存。  
需要注意的是对象池并不是只要存储[0-9999]的整数就可以工作。当设置maxmemory并启用LRU相关淘汰策略如:volatile-lru，allkeys-lru时，Redis禁止使用共享对象池。

```bash
redis> set key1 99
OK
redis> object refcount key1
(integer) 2 //使用了对象共享,引用数为2
redis> config set maxmemory-policy volatile-lru
OK //开启LRU淘汰策略
redis> set key2 99
OK 
redis> object refcount key2
(integer) 3 //使用了对象共享,引用数变为3
redis> config set maxmemory 1GB
OK //设置最大可用内存
redis> set key3 99
OK //设置key3=99
redis> object refcount key:3
(integer) 1 //未使用对象共享,引用数为1
redis> config set maxmemory-policy volatile-ttl
OK //设置非LRU淘汰策略
redis> set key4 99
OK //设置key4=99
redis> object refcount key4
(integer) 4 //又可以使用对象共享,引用数变为4
```

#### 为什么开启maxmemory和LRU淘汰策略后对象池无效?  
LRU算法需要获取对象最后被访问时间，以便淘汰最长未访问数据，每个对象最后访问时间存储在redisObject对象的lru字段。  
对象共享意味着多个引用共享同一个redisObject，这时lru字段也会被共享，导致无法获取每个对象的最后访问时间。如果没有设置maxmemory，直到内存被用尽Redis也不会触发内存回收，所以共享对象池可以正常工作。

综上所述，共享对象池与maxmemory+LRU策略冲突，使用时需要注意。  
对于ziplist编码的值对象，即使内部数据为整数也无法使用共享对象池，因为ziplist使用压缩且内存连续的结构，对象共享判断成本过高。

#### 为什么只有整数对象池？
首先整数对象池复用的几率最大，其次对象共享的一个关键操作就是判断相等性，Redis之所以只有整数对象池，是因为整数比较算法时间复杂度为O(1)，只保留一万个整数为了防止对象池浪费。  
如果是字符串判断相等性，时间复杂度变为O(n)，特别是长字符串更消耗性能(浮点数在Redis内部使用字符串存储)。  
对于更复杂的数据结构如hash, list等，相等性判断需要O(n2)。  
对于单线程的Redis来说，这样的开销显然不合理，因此Redis只保留整数共享对象池。

## 四. 字符串优化
字符串对象是Redis内部最常用的数据类型。所有的键都是字符串类型，值对象数据除了整数之外都使用字符串存储。

#### 1.字符串结构
Redis没有采用原生C语言的字符串类型而是自己实现了字符串结构，内部简单动态字符串(simple dynamic string)，简称SDS。
Redis自身实现的字符串结构有如下特点:

- O(1)时间复杂度获取：字符串长度，已用长度，未用长度。
- 可用于保存字节数组，支持安全的二进制数据存储。
- 内部实现空间预分配机制，降低内存再分配次数。
- 惰性删除机制，字符串缩减后的空间不释放，作为预分配空间保留。

#### 2.预分配机制
因为字符串(SDS)存在预分配机制，要小心预分配带来的内存浪费。
- 第一次创建字符串，len属性等于数据实际大小，free等于0，不做预分配。
- 修改后如果已有free空间不够且数据小于1M，每次预分配一倍容量。如原有len=60byte，free=0，再追加60byte，预分配120byte，总占用空间:60byte+60byte+120byte+1byte。
- 修改后如果已有free空间不够且数据大于1MB，每次预分配1MB数据。如原有len=30MB，free=0，当再追加100byte ,预分配1MB，总占用空间:1MB+100byte+1MB+1byte。

> 尽量减少字符串频繁修改操作如append，setrange, 改为直接使用set修改字符串，降低预分配带来的内存浪费和内存碎片化。

#### 3.字符串重构
不一定把每份数据作为字符串整体存储，像json这样的数据可以使用hash结构，使用二级结构存储也能帮我们节省内存。  
同时可以使用hmget,hmset命令支持字段的部分读取修改，而不用每次整体存取。

分别使用字符串和hash结构测试内存表现，如下表所示。

数据量 | key	  | 存储类型 |   value     | 配置	                      | used_mem
----- | ----- | ------- | ----------- | ------------------------- | -------
200W  | 20字节 | string  |  json字符串  | 默认                      | 612.62M
200W  | 20字节 | hash    | key-value对 | 默认                      | 默认1.88GB
200W  | 20字节| 	hash    | key-value对 | hash-max-ziplist-value:66 | 535.60M

根据测试结构，第一次默认配置下使用hash类型，内存消耗不但没有降低反而比字符串存储多出2倍，而调整hash-max-ziplist-value=66之后内存降低为535.60M。  
因为json的videoAlbumPic属性长度是65，而hash-max-ziplist-value默认值是64，Redis采用hashtable编码方式，反而消耗了大量内存。  
调整配置后hash类型内部编码方式变为ziplist，相比字符串更省内存且支持属性的部分操作。

## 五. 编码优化
#### 1.了解编码
所谓编码就是具体使用哪种底层数据结构来实现。  
编码不同将直接影响数据的内存占用和读写效率。  
使用`object encoding key`命令获取编码类型。
```bash
redis> set str:1 hello
OK
redis> object encoding str:1
"embstr" // embstr编码字符串
redis> lpush list:1 1 2 3
(integer) 3
redis> object encoding list:1
"ziplist" // ziplist编码列表
```

Redis针对每种数据类型(type)可以采用至少两种编码方式来实现，下表表示type和encoding的对应关系。

类型   | 编码方式 | 数据结构
----  | ------- | ---- 
<br>string|	raw<br>embstr<br>int | 动态字符串编码<br>优化内存分配的字符串编码<br>整数编码
hash  |hashtable<br>ziplist | 散列表编码<br>压缩列表编码
<br>list  |linkedlist<br>ziplist<br>quicklist|双向链表编码<br>压缩列表编码<br>3.2版本新的列表编码
set	|hashtable<br>intset |散列表编码<br>整数集合编码
zset| skiplist<br>ziplist  |跳跃表编码<br>压缩列表编码

#### 2.控制编码类型
编码类型转换在Redis写入数据时自动完成，这个转换过程是不可逆的，转换规则只能从小内存编码向大内存编码转换。例如：

```bash
redis> lpush list:1 a b c d
(integer) 4 //存储4个元素
redis> object encoding list:1
"ziplist" //采用ziplist压缩列表编码
redis> config set list-max-ziplist-entries 4
OK //设置列表类型ziplist编码最大允许4个元素
redis> lpush list:1 e
(integer) 5 //写入第5个元素e
redis> object encoding list:1
"linkedlist" //编码类型转换为链表
redis> rpop list:1
"a" //弹出元素a
redis> llen list:1
(integer) 4 // 列表此时有4个元素
redis> object encoding list:1
"linkedlist" //编码类型依然为链表，未做编码回退
```
list-max-ziplist-entries参数用来决定列表长度在多少范围内使用ziplist编码。当然还有其它参数控制各种数据类型的编码

类型|编码|决定条件
--- | --- | ---
hash|ziplist|1.value最大空间(字节)<=hash-max-ziplist-value<br>2.field个数<=hash-max-ziplist-entries
hash|hashtable|不满足以上任意条件
list|ziplist|1.value最大空间(字节)<=list-max-ziplist-value<br>2.链表长度<=list-max-ziplist-entries
list|linkedlist|不满足以上任意条件
list|quicklist|3.2版本新编码: 废弃list-max-ziplist-entries和list-max-ziplist-entries<br> 使用新配置: list-max-ziplist-size:表示最大压缩空间或长度,最大空间使用[-5-1]范围配置，默认-2表示8KB,正整数表示最大压缩长度<br>list-compress-depth:表示最大压缩深度，默认=0不压缩
set|intset|1.元素必须为整数<br>2.集合长度<=set-max-intset-entries
set	|hashtable|不满足以上任意条件
zset|ziplist|1.value最大空间(字节)<=zset-max-ziplist-value<br>2.有序集合长度<=zset-max-ziplist-entries
zset|skiplist|不满足以上任意条件


理解编码转换流程和相关配置之后，可以使用config set命令设置编码相关参数来满足使用压缩编码的条件。对于已经采用非压缩编码类型的数据如hashtable,linkedlist等，设置参数后即使数据满足压缩编码条件，Redis也不会做转换，需要重启Redis重新加载数据才能完成转换。

>1）针对性能要求较高的场景使用ziplist，建议长度不要超过1000，每个元素大小控制在512字节以内。  
>2）命令平均耗时使用info Commandstats命令获取，包含每个命令调用次数，总耗时，平均耗时，单位微秒。


## 六 控制key的数量
当使用Redis存储大量数据时，通常会存在大量键，过多的键同样会消耗大量内存。使用Redis时不要进入一个误区，大量使用get/set这样的API，把Redis当成Memcached使用。对于存储相同的数据内容利用Redis的数据结构降低外层键的数量，也可以节省大量内存。

**hash分组控制键规模测试**

数据量|key大小|value大小|string类型占用内存|hash-ziplist类型占用内存|内存降低比例|string:set平均耗时|hash:hset平均耗时
--- | ---- |------ | ------- | ------- | --- | ----- | ------
200w|20byte|512byte|1392.64MB|1000.97MB|28.1%|2.13微秒|21.28微秒
200w|20byte|200byte|596.62MB|399.38MB|33.1%|1.49微秒|16.08微秒
200w|20byte|100byte|382.99MB|211.88MB|44.6%|1.30微秒|14.92微秒
200w|20byte|50byte|291.46MB|110.32MB|62.1%|1.28微秒|13.48微秒
200w|20byte|20byte|246.40MB|55.63MB|77.4%|1.10微秒|13.21微秒
200w|20byte|5byte|199.93MB|24.42MB|87.7%|1.10微秒|13.06微秒

**以上测试数据说明**

- 同样的数据使用ziplist编码的hash类型存储比string类型节约内存
- 节省内存量随着value空间的减少，越来越明显。
- hash-ziplist类型比string类型写入耗时，但随着value空间的减少，耗时逐渐降低。
- 使用hash重构后节省内存量效果非常明显，特别对于存储小对象的场景，内存只有不到原来的1/5。

**下面分析这种内存优化技巧的关键点：**

- hash类型节省内存的原理是使用ziplist编码，如果使用hashtable编码方式反而会增加内存消耗。
- ziplist长度需要控制在1000以内，否则由于存取操作时间复杂度在O(n)到O(n2)之间，长列表会导致CPU消耗严重，得不偿失。
- ziplist适合存储的小对象，对于大对象不但内存优化效果不明显还会增加命令操作耗时。对于大量小对象的存储场景，非常适合使用ziplist编码的hash类型控制键的规模来降低内存。
- 需要预估键的规模，从而确定每个hash结构需要存储的元素数量。
- 根据hash长度和元素大小，调整hash-max-ziplist-entries和hash-max-ziplist-value参数，确保hash类型使用ziplist编码。

**带来的问题**

- hash重构后所有的键无法再使用超时(expire)和LRU淘汰机制自动删除，需要手动维护删除。
- 对于大对象，如1KB以上的对象。使用hash-ziplist结构控制键数量。

不过瑕不掩瑜，对于大量小对象的存储场景，非常适合使用ziplist编码的hash类型控制键的规模来降低内存。

>使用ziplist+hash优化keys后，如果想使用超时删除功能，可以存储每个对象写入的时间，再通过定时任务使用hscan命令扫描数据，找出hash内超时的数据项删除即可。
