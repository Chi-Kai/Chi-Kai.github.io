---
author : "KC"
type : "post"
tags :
    - 源码剖析 
    - Redis
categories :
    - 折腾
title: "Redis源码剖析(一)"
description: 更新中
date: 2022-05-22
draft: false
toc: true
---
# 基础数据结构部分
## 动态字符串 SDS 
### 前置知识

由于我对C语言没有深入了解，有很多知识点会在前面补充。

1. __attribute__ ((__packed__)): 对齐优化

	__attribute__((args)) 是GNU C的一个机制，可以通过编译器来修饰结构体，函数等。

	现代计算机中内存空间都是按照字节（byte）划分的，从理论上讲似乎对任何类型的变量的访问可以从任何地址开始，但实际情况是在访问特定变量的时候经常在特定的内存地址访问，这就需要各类型数据按照一定的规则在空间上排列，而不是顺序地一个接一个地排放，这就是对齐.
  
  	为了提高效率，计算机从内存中取数据是按照一个固定长度的。以32位机为例，它每次取32个位，也就是4个字节（每字节8个位)。字节对齐有什么好处？以int型数据为例，如果它在内存中存放的位置按4字节对齐，也就是说1个int的数据全部落在计算机一次取数的区间内，那么只需要取一次就可以了

	使用__packed__参数是表示，使用原来的地址空间，编译时不要字节对齐，这样用时间换空间，使得结构体紧密。

	详细的用法见 [机制详解](https://blog.csdn.net/weaiken/article/details/88085360)。

2. uint8_t uint16_t ... size_t 使用

	后面加_t表示是一个typedef 定义的类型，本质是原有类型。这样做是为了更好的跨平台移植，因为不同的平台中int,long 这些基础类型可能占用的字节不同，这对于一些对内存严格要求的库造成不便。使用uint8_t 等类型，在不同平台上都代表占一个字节8位，便于程序的实现。

	同理 size_t 也是用来保持跨平台移植性。可以是unsigned int unsigned char unsigned long等等，取决于实现，size_t = typeof(sizeof(X))。

3. static inline
   
	 头文件中很多函数使用了static inline 关键字，inline 建议编译器将函数作为一个宏内联，这样可以减少函数调用时的堆栈消耗，提高性能。但是编译器不一定会内联函数，这时候static可以保证这个函数是仅在本文件可见，避免重复包含冲突。

### 数据结构

这里以sdshdr8为例

```c
// 一个字节 8位
// __attribute__ ((__packed__)) 用来告诉编译器取消结构在编译中的优化对齐，按照实际占用。
// 因为内存是按照2的倍数读取的，否则可能读两次,速度变慢。这里是用时间换空间
// 保证整个结构体的空间紧密
struct __attribute__ ((__packed__)) sdshdr8 {
    // buf 中已经使用的字节数
    uint8_t len; /* used */
    // 去掉头和null结束符，已经分配的字节数=有效长度+数据长度
    uint8_t alloc; /* excluding the header and null terminator */ 
    // 8位，只用前三位
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    // 柔性数组，没有分配之前不占内存
    char buf[];
};

```

#### 二进制安全
 
 	什么是二进制安全？通俗地讲，C语言中，用“\0”表示字符串的结束，如果字符串中本身就有“\0”字符，字符串就会被截断，即非二进制安全；若通过某种机制，保证读写字符串时不损害其内容，则是二进制安全。在网络报文中常常需要二进制安全。

	sds使用 len 来控制字符串长度，而不是使用"\0",保障了二进制安全。 

#### 极致的内存使用
   
   - 对于不同的长度的字符串有不同的结构，上面的sdshdr8 表示长度为8位的字符串，还有sdshdr16/32/64。保证不会因为字符串过小而额外浪费字节，也不会因为字符串过长而频繁扩容。
	- 结构体紧密，放弃对齐优化。在前置说明了结构体使用编译器参数packed来放弃优化，**用时间换空间**。
	- flag。使用一个unsigned char,8位的小端三位来表示结构体的性质(5/8/16/32/64)。
	- 变长数组（柔性数组）。柔性数组成员（flexible array member），也叫伸缩性数组成员，**只能被放在结构体的末尾**。包含柔性数组成员的结构体，通过malloc函数为柔性数组动态分配内存。之所以用柔性数组存放字符串，是因为柔性数组的地址和结构体是连续的，这样查找内存更快（因为不需要额外通过指针找到字符串的位置）；可以很方便地通过柔性数组的首地址偏移得到结构体首地址，进而能很方便地获取其余变量。

#### 与c字符串函数
  
	始终将buf指针暴露给上层，可以和c字符串函数切合。同时可以很容易地通过减去一个sdshdr大小偏移到结构体首部来调用结构体属性。
		
	如下面的宏定义:

	```c
	#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
	#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
	```
### 基本操作

#### 创建

```c
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    // 因为通常总是有空字符串，而使用type5每增加一次就需要扩容，所以直接使用type 8 ---- 为什么不把type5直接删了？
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
    size_t usable;

    // initlen 是buf中实际装的大小，hdrlen是sds header大小，+1 是\0终止符
    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    //malloc_usable还是调用tyymalloc_usable
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    // 指向buf
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    usable = usable-hdrlen-1;
    // 可能申请的超过类型MaxSize
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
    // ...
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

1. 首先根据申请的初始大小来确定类型，通过类型可以确定hdr大小，然后来申请空间。
2. 这里使用的s_trymalloc_usable与s_malloc_usable都是文件zmallo.c实现的内存管理函数，后面会专门讲解。这里只用知道它会申请前一个参数大小的空间，并且将空间大小赋值给后一个参数usable。
3. 得到空间的首地址，加上头大小得到buf地址s,s[-1] 得到类型指针fp,usable减去头大小hdrlen和类型大小1得到实际可用大小。
4. 根据类型来构建一个sds结构体，最后返回是buf的指针，补上终止符'\0'。这里使用的是一个宏，可以借鉴这种写法，一个经常使用的操作，如果写成函数，会增加堆栈调度消耗，写成宏可以提高性能，代价是编译后的文件大小会增加。

这是sds创建的底层实现，实际使用的是上层的封装，只是对这个函数的封装调用。

#### 销毁

有两种方法，一种是直接销毁:

```c
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```
一种是仅仅将sds的len标记为0,但是实际的buf并不会释放，而是等待覆写。这样可以优化性能。

#### 扩容

```c
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    reqlen = newlen = (len+addlen);
    // 这里是防止溢出
    assert(newlen > len);   /* Catch size_t overflow */
    if (greedy == 1) {
        // SDS_MAX_PREALLOC 是 1MB
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;
    }

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    if (oldtype==type) {
        // 和原类型相同，则不用释放内存，直接将buf扩容即可
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        // 类型改变,需要重新申请内存，原内存释放
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    usable = usable-hdrlen-1;
    // type 是通过newlen判断得到的，而usable 是 hdrlen + newlen + 1 可能出现超出的情况
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}
```
1. 首先判断newlen加上len是否超出可用的大小avail,没超就不扩容。
2. 和之前不同，这一版本加入了greedy参数，来调节扩容策略，当greedy为1时，每次会扩大的比所需要的更多，这样可以减少扩容频率。而greedy为0时，就是节约内存
3. greedy为1时启用此策略: 如果newlen小于1MB,每次扩容二背，大于1MB时每次增加1MB。（每次2倍内存很快就耗尽了）
4. 这里根据新的newlen来确定类型，如果类型不变，只需要扩展buf数组，而类型改变的话就需要重新申请内存。



## 跳表zskiplist

对应的代码在 server.h 和 t_zset.c。

跳表可以看作链表加上都多层索引，一般每两个

![skiplist](https://s2.loli.net/2022/11/25/87fPGrURAgdHzn3.jpg)

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele; // 存储字符串类型的数据
    double score; // 储存排序的分值
    struct zskiplistNode *backward; // 后向指针 头节点和第一个节点都为NULL
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 指向本层下一个节点
        unsigned long span; // 跨度 指向本层下一个节点中间跨越的节点个数
    } level[]; // 柔性数组，未分配内存时不占空间。初始化时，level 随机分配1~32
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail; 
    unsigned long length; // 除了头节点以外节点总数
    int level; // 跳表的高度
} zskiplist;

```
用的图片是网图 [链接](https://zhuanlan.zhihu.com/p/26499803)，其中obj一般为sds，在Redis6中已经改为sds ele。

可以从图和代码很清晰地看出跳跃表的结构。

跳跃表是Redis有序集合的底层实现方式之一，所以每个节点的ele存储有序集合的成员member值，score存储成员score值。所有节点的分值是按从小到大的方式排序的，当有序集合的成员分值相同时，节点会按member的字典序进行排序。

通过跳跃表结构体的属性我们可以看到，程序可以在O(1)的时间复杂度下，快速获取到跳跃表的头节点、尾节点、长度和高度。

### 创建

Redis通过zslRandomLevel函数随机生成一个1～32的值，作为新建节点的高度，值越大出现的概率越低。节点层高确定之后便不会再修改。生成随机层高的代码如下。

```c
// ZSKIPLIST 为 0.25
int zslRandomLevel(void) {
    static const int threshold = ZSKIPLIST_P*RAND_MAX;
    int level = 1;
    while (random() < threshold)
        level += 1;
    // 这里 ZSKIPLIST_MAXLEVEL 为32 
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

当p=0.25时，跳跃表节点的期望层高为1/(1-0.25)≈1.33。

下面是创建函数
```c
/* Create a skiplist node with the specified number of levels.
 * The SDS string 'ele' is referenced by the node after the call. */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
		    // level 柔性数组加上头大小 来申请内存
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}

/* Create a new skiplist. */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
		// 头节点的level是最大层数
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```
头节点是一个特殊的节点，不存储有序集合的member信息。头节点是跳跃表中第一个插入的节点，其level数组的每项forward都为NULL, span值都为0

### 插入

```c
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    // 记录每层所能到达的最右边节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // 记录每层从header到update[i}所需的步长
    unsigned long rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    // 判断score 是不是NAN
    serverAssert(!isnan(score));
    x = zsl->header;
    // 从最高层索引开始遍历
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        // 当在最高层时，先将rank赋值为0，先假设
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        
        // 在第i层一直向前移动比较,因为是按照score 从小到大排列的
        // 找到这层大于插入score 的位置然后下移
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            // 更新总的span
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        // 记录这层的终点
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    // zslInsert不能应用在插入节点已经存在的情况下。
    // 所以不用检查存在
    
    //为插入节点计算随机层数
    level = zslRandomLevel();
    //大于原来层高的部分，只需要调整header就行。
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            // 为啥是这个？可能是用来占位
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        // 插入到每层最右侧能到达的节点之后
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        // 插入节点每层的span更新,这个看下图
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
    
    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

下图来源于 [链接](https://zhuanlan.zhihu.com/p/56941754)

![](https://s2.loli.net/2022/11/26/FHEMRScINYyX72U.png)

以节点19插入为例，其中
黑色箭头的表示的跨度为update[i]->level[i].span
蓝色箭头表示的跨度为rank[0] - rank[i]即节点19在level_0的update[0]为11，
在level_1的update[1]为7，rank[0] - rank[i]为节点7与节点11之间的跨度
绿色箭头表示的跨度为节点19到节点37的span


### 删除

首先查找到对应的节点，将每层最右边到达的节点记录下来，对应的update。
辅助函数:

```c
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    // 调整对应的span和forward
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }

    // 调整level 
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```
删除函数:

```c
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;
    
    //查找位置
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        // 保存每层最右边的节点
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    // 可能同一个score有多个ele
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```

## 压缩列表

具体的实现在ziplist.h和ziplist.c

压缩列表是Redis中一种高效利用内存的数据结构，用来储存字符串和数字，它的push和pop操作都是O(1)。Redis的有序集合、散列和列表都直接或者间接使用了压缩列表。当有序集合或散列表的元素个数比较少，且元素都是短字符串时，Redis便使用压缩列表作为其底层数据存储结构。列表使用快速链表（quicklist）数据结构存储，而快速链表就是双向链表与压缩列表的组合。

```c
// ziplist 结构
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```
这里的所有结构都是按照小端存储。

- zlbytes: 压缩列表的字节长度，占4个字节，因此压缩列表最多有$2^{32}-1$个字节。这个设计是为了resize时不必遍历整个列表
- zltail:  压缩列表尾元素相对于压缩列表起始地址的偏移量，占4个字节，这个设计可以使pop操作不必要遍历全部。
- zllen: 压缩列表的元素个数，占2个字节。zllen无法存储元素个数超过65535（$2^{16}-1$）的压缩列表，必须遍历整个压缩列表才能获取到元素个数。
- zlend: 压缩列表的结尾，占1个字节，恒为0xFF。

这里可以清楚地感受到C语言对内存的掌控，通过指针位移来获取结构信息。这里使用宏又是C语言的一个特色，比起inline只是建议编译器内联，宏真正是内联，对于一些细小而频繁的操作提高了性能。

```c
/* Return total bytes a ziplist is composed of. */
#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))

/* Return the offset of the last item inside the ziplist. */
#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t*)((zl)+sizeof(uint32_t))))

/* Return the length of a ziplist, or UINT16_MAX if the length cannot be
 * determined without scanning the whole ziplist. */
#define ZIPLIST_LENGTH(zl)      (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))

/* The size of a ziplist header: two 32 bit integers for the total
 * bytes count and last item offset. One 16 bit integer for the number
 * of items field. */
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))

/* Size of the "end of ziplist" entry. Just one byte. */
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))

/* Return the pointer to the first entry of a ziplist. */
#define ZIPLIST_ENTRY_HEAD(zl)  ((zl)+ZIPLIST_HEADER_SIZE)

/* Return the pointer to the last entry of a ziplist, using the
 * last entry offset inside the ziplist header. */
#define ZIPLIST_ENTRY_TAIL(zl)  ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))

/* Return the pointer to the last byte of a ziplist, which is, the
 * end of ziplist FF entry. */
#define ZIPLIST_ENTRY_END(zl)   ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl))-ZIPLIST_END_SIZE)
```
对于 <entry> 结构如下:

```c
<prevlen> <encoding> <entry-data>
```
previous_entry_length字段表示前一个元素的字节长度，占1个或者5个字节，当前一个元素的长度小于254字节时，用1个字节表示；当前一个元素的长度大于或等于254字节时，用5个字节来表示。而此时previous_entry_length字段的第1个字节是固定的0xFE，后面4个字节才真正表示前一个元素的长度。假设已知当前元素的首地址为p，那么p-previous_entry_length就是前一个元素的首地址，从而实现压缩列表从尾到头的遍历。

encoding字段表示当前元素的编码，即content字段存储的数据类型（整数或者字节数组），数据内容存储在content字段。为了节约内存，encoding字段同样长度可变。

![](https://s2.loli.net/2022/11/26/6gMtVLmx9udXpS3.png)

Redis使用宏来表示

```c
#define ZIP_STR_MASK 0xc0
#define ZIP_INT_MASK 0x30
#define ZIP_STR_06B (0 << 6)
#define ZIP_STR_14B (1 << 6)
#define ZIP_STR_32B (2 << 6)
#define ZIP_INT_16B (0xc0 | 0<<4)
#define ZIP_INT_32B (0xc0 | 1<<4)
#define ZIP_INT_64B (0xc0 | 2<<4)
#define ZIP_INT_24B (0xc0 | 3<<4)
#define ZIP_INT_8B 0xfe
``` 
**这里使用位运算来代表类型**，既节省了内存又提高了性能。

### 结构

```c
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
```
对于压缩列表的任意元素，获取前一个元素的长度、判断存储的数据类型、获取数据内容都需要经过复杂的解码运算。解码后的结果应该被缓存起来，为此定义了结构体zlentry，用于表示解码后的压缩列表元素。

 

```c
static inline void zipEntry(unsigned char *p, zlentry *e) {
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    assert(e->lensize != 0); /* check that encoding was valid. */
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}
```

