最外层数据结构redisDb



redisDb源码

```c
typedef struct redisDb {
    dict *dict;     /* 表示数据库的键空间，存储所有的键值对。这个字典结构包含了数据库中所有的键以及它们对应的值 */
    dict *expires;  /* 存储每个设置了超时时间的键及其对应的超时时间。当键过期时，这些键会从数据库中删除 */
    /*存储那些有客户端在等待数据的键。例如，当客户端发送 BLPOP 命令等待列表中的元素时，这些键会被记录在这个字典中*/
    dict *blocking_keys; 
    /*存储那些已经收到 PUSH 操作的被阻塞的键。当一个键的值被 PUSH 新数据时，这个键会从blocking_keys移动到ready_keys中*/
    dict *ready_keys;          
    /*存储在 MULTI/EXEC 事务中被 WATCH 的键。在事务执行时，如果这些键发生变化，事务将被中止*/
    dict *watched_keys;        
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    /*用于追踪活动过期周期的游标。在 Redis 的过期机制中，这个游标用于迭代检查哪些键已经过期并需要删除 */
    unsigned long expires_cursor; 
    /* 存储那些需要逐个尝试进行碎片整理的键名列表。碎片整理可以帮助优化内存使用 */
    list *defrag_later;         
} redisDb;
```



dict源码

```c
typedef struct dict {
    dictType *type; //dict类型，内置不同的hash函数
    void *privdata; //指向私有数据的指针，在做特殊hash运算时用
    dictht ht[2]; //一个包含两个哈希表的数组，存储具体值的数组结构，ht[1]用于渐进式rehash
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```



dictht源码

```c
typedef struct dictht {
    dictEntry **table; //dictEntry数组，保存的是指向entry的指针
    unsigned long size; //hash表大小
    unsigned long sizemask; //hash表 size - 1
    unsigned long used; //已经使用的大小，等于entry个数
} dictht;
```



dictEntry源码

```c
typedef struct dictEntry {
    void *key;  //key值
    union {     // value值
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; //下一个entry的指针，使用链地址法解决hash冲突
} dictEntry;
```



![image-20240612172429814](/Users/wangsen/Library/Application Support/typora-user-images/image-20240612172429814.png)



**dictht的扩容机制**



扩容时机

在put数据时，检查负载因子(LoadFctor = used / size 已使用大小和总大小的比值)，满足以下情况时触发扩容

+ LoadFctor >= 1，并且没有子进程在做持久化操作时
+ LoadFctor > 5

当存在子进程在做持久化操作时，认为当前cpu主要为持久化服务，LoadFctor大于1也不会造成太大的性能影响故先不进行扩容，但是当其大于5的时候，认为数组已经满负荷，迫切需要进行扩容

```c
/* Expand the hash table if needed */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if ((dict_can_resize == DICT_RESIZE_ENABLE &&
         d->ht[0].used >= d->ht[0].size) ||
        (dict_can_resize != DICT_RESIZE_FORBID &&
         d->ht[0].used / d->ht[0].size > dict_force_resize_ratio))
    {
      	//扩容函数
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

Dict除了扩容以外，每次删除元素时，也会对负载因⼦做检查，当LoadFactor ＜ 0.1 时，会做哈希表收缩

```c
if (dictDelete((dict*)o->ptr, field) == C_OK) {
       deleted = 1;
       /* Always check if the dictionary needs a resize after a delete. */
       if (htNeedsResize(o->ptr)) dictResize(o->ptr);
 }
```

```c
int htNeedsResize(dict *dict) {
    long long size, used;

    size = dictSlots(dict);
    used = dictSize(dict);
    return (size > DICT_HT_INITIAL_SIZE &&
            (used*100/size < HASHTABLE_MIN_FILL));
}
```



扩容流程







Redis中的任意数据类型都会被封装为⼀个RedisObject，它表示 Redis 中存储的对象的值

```c
typedef struct redisObject {
    unsigned type:4; /* 对象的类型，例如字符串、列表、集合、有序集合、哈希等 */
    unsigned encoding:4; /* 对象的编码方式，例如原始字符串、整数、压缩列表等 */
    unsigned lru:LRU_BITS; /* 对象的 LRU（Least Recently Used）时间戳，用于实现对象的过期和内存管理 */
    int refcount; /* 引用计数，表示有多少地方引用了这个对象。当引用计数为零时，可以安全地删除这个对象 */
    void *ptr; /* 指向存放实际数据的地址 */
} robj;
```



5.0版本引入listpack，7.0正式替换ziplist

ZipList虽然节省内存,但申请内存必须是连续空间,如果内存占用较多,申请内存效率很低。怎么办？

限制ZipList的长度和entry大小

存储大量数据,超出了ZipList最佳的上限该怎么办？

创建多个ZipList分片存储数据

数据拆分后比较分散,不方便管理和查找,这多个ZipList如何建立联系？

在3.2版本引入了新的数据结构QuickList,它是一个双端链表,只不过链表中的每个节点都是一个ZipList

![image-20240612093457987](/Users/wangsen/Library/Application Support/typora-user-images/image-20240612093457987.png)

为了避免QuickList中的每个ZipList中entry过多，Redis提供了一个配置项:list-max-ziplist-size来限制

如果值为正,则代表ZipList的允许的entry个数的最大值
如果值为负,则代表ZipList的最大内存大小,分5种情况:(其默认值为 -2)

```c
-1:每个ZipList的内存占用不能超过4kb
-2:每个ZipList的内存占用不能超过8kb
-3:每个ZipList的内存占用不能超过16kb
-4:每个ZipList的内存占用不能超过32kb
-5:每个ZipList的内存占用不能超过64kb
```



ziplist压缩配置:list-compress-depth 0: 表示一个quicklist两端不被压缩的节点个数
这里的节点是指quicklist双向链表的节点，而不是指ziplist里面的数据项个数，参数list-compress-depth的取值含义如下:

```c
0: 是个特殊值,表示都不压缩。这是Redis的默认值
1: 表示quicklist两端各有1个节点不压缩,中间的节点压缩
2: 表示quicklist两端各有2个节点不压缩,中间的节点压缩
3: 表示quicklist两端各有3个节点不压缩,中间的节点压缩
依此类推…
```



**quicklist源码**

```c
typedef struct quicklist {
    quicklistNode *head;       /* 头节点 */
    quicklistNode *tail;       /* 尾节点 */
    unsigned long count;       /* 所有ziplists中entry的总数 */
    unsigned long len;         /* quicklistNodes 的数量 */
    int fill : QL_FILL_BITS;   /* 每个节点的填充因子，控制 ziplist 的 entry 个数，默认值为 -2 */
    unsigned int compress : QL_COMP_BITS; /* 不压缩的末端节点深度；0 表示关闭压缩 */
    unsigned int bookmark_count: QL_BM_BITS; /* 书签的数量 */
    quicklistBookmark bookmarks[]; /* 书签数组，用于快速定位 */
} quicklist;
```

**quicklistNode源码**

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;  /* 指向前一个节点的指针 */
    struct quicklistNode *next;  /* 指向下一个节点的指针 */
    unsigned char *zl;           /* 指向 ziplist 数据的指针 */
    unsigned int sz;             /* ziplist 的大小，以字节为单位 */
    unsigned int count : 16;     /* ziplist 中元素的数量 */
    unsigned int encoding : 2;   /* 编码模式，RAW==1 表示未压缩的 ziplist，LZF==2 表示 LZF 压缩模式 */
    unsigned int container : 2;  /* 容器类型，NONE==1 或 ZIPLIST==2 */
    unsigned int recompress : 1; /* 标记该节点是否之前被压缩过 */
    unsigned int attempted_compress : 1; /* 标记该节点是否尝试压缩但失败（因为数据太小） */
    unsigned int extra : 10;     /* 预留的额外位，将来可能使用 */
} quicklistNode;
```

优点

在ziplist的基础上进一步优化了内存占用和申请效率的问题
节点采用ZipList，并且可以进行压缩，解决了传统链表的内存占用问题。控制了ZipList大小,解决连续内存空间申请效率问题

skipList

跳表是可以实现二分查找的有序链表

skiplist是一种以空间换取时间的结构，由于链表,无法进行二分查找，因此借鉴数据库索引的思想：提取出链表中关键节点(索引），先在关键节点上查找，再进入下层查找。提取多层关键节点，就形成了跳表

![image-20240612123125338](/Users/wangsen/Library/Application Support/typora-user-images/image-20240612123125338.png)



源码

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level; //最大的索引层级，默认为1
} zskiplist;
```

```c
typedef struct zskiplistNode {
    sds ele;    //节点存储的值
    double score; //节点分数，用于排序、查找
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span; //索引跨度
    } level[];  //多级索引数组
} zskiplistNode;
```



![WeChat0ec56074f68a6b44d75ccb7d8d132d93_enhanced](/Users/wangsen/Downloads/WeChat0ec56074f68a6b44d75ccb7d8d132d93_enhanced.png)



特点

1. 跳跃表是一个双向链表，每个节点都包含score(分数)和ele(内容)值
2. 节点按照score值排序，score值一样则按照ele字典排序
3. 每个节点都可以包含多层指针，层数是1到32之间的随机数
4. 不同层指针到下一个节点的跨度不同，层级越高，跨度越大