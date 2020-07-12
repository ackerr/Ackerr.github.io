---
title: 深入浅出Redis之sds
date: 2020-07-12 23:25:24
tags:
  - redis
  - sds
categories:
  - redis
---

> 最近在看《Redis的设计与实现》，感觉书还不错，但由于书中讲解的Redis版本较低，有些地方与现版本有出入，所以通过简单浏览源码，加深理解。本文代码使用[Redis6.0](https://github.com/redis/redis/tree/6.0)

SDS，全称Simple Dynamic String，作为Redis中的字符串表示，主要作用于Redis中字符串对象StringObject的底层实现，以及Redis内部作为char*类型的替代品。具体sds源码实现在sds.c和sds.h文件中。

## 数据结构

查看sds.h文件，不难发现对sds的类型定义，其实就是char *的一个alias。

```c
typedef char *sds;
```

尽管C语言字符串本质上也是char *，不过与之不同的是，一个sds的完整结构，分为两块部分，sdshdr结构和字符数组。以下为sdshdr结构体：

```c
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
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

/* flags常量 */
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

sds针对不同的字符串长度，为了节省内存，定义了五种类型的header，其中sdshdr5较为特殊，与其他header结构不同，它并没有alloc和len字段，所以flags字段被分为两部分，低三位作为header类型，高五位记录字符串长度。因为不具备alloc字段，所以sdshdr5不会预分配空间，如果字符串动态增加，则需要重新分配内存，大部分场景不太会使用sdshdr5。其余4个header的结构类似，字段含义如下

- len： 字符串真实长度
- alloc：字符串当前可使用的最大容量
- flags：占一字节，前三位表示header类型

其中结构体中，有几个特殊的点

- `__attribute__ ((__packed__))`表示编译器不需要对结构体进行内存对齐，从而使字段内存连续
- `char buf[]`，未指定长度且定义在结构体的最后一个字段，是一个柔性数组，表示flags字段后是一个字符数组，而柔性数组buf[]只作为一个符号地址存在，通过sizeof 返回的结构大小不包括柔性数组的内存大小

通过源码中定义的宏以及sdslen方法不难看出这两点

```c
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3

#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)

static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);  /* sdshdr5 len保存在flags的高五位中*/
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```

> ”##“ 连接符，预处理运算符，例如SDS_HDR(8,s)宏展开后是((struct sdshdr8 *)((s)-(sizeof(struct sdshdr8))))

其中因为buf[]是柔性数组，sizeof(struct sdshdr8)不会包含数组大小，SDS_HDR即表示字符串向前偏移结构体的大小，从而获得header起始位置的指针。而flags则可以通过s向前偏移一个字节获得，即`flags=s[-1]`

综上，不难看出sds结构体中的header部分，实际存储在字符数组之前，如下图所示

```
+---------------------+-------------------------------+-----------+
| len | alloc | flags | Binary safe C alike string... | Null term |
+---------------------+-------------------------------+-----------+
          |           |
          `->Header   `-> Pointer returned to the user.
```



## SDS的一些操作

### 创建

```c
/* 通过sdsnew函数为字符数组创建一个sds */
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}

sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);  /* 根据所给字符串长度给出适合的sds类型 */
    /* 长度为0时，由于可能会进行追加操作，而SDS_TYPE_5不适合追加数据，所以sds类型为SDS_TYPE_8*/
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type); /* 根据sds类型返回对应的sdshdr大小 */
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1); /* 申请内存，需要为终止符\0额外申请1字节*/
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1); /* 分配内存 */
    s = (char*)sh+hdrlen; /* 内存向右偏移sdshdr大小，获取数组指针 */
    fp = ((unsigned char*)s)-1; /* 数组指针向左偏移一位，获取flags指针 */
    switch(type) {  /* 根据sds类型，设置sdshdr alloc len flags 值 */
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        ...
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';  /* 字符结尾设置\0终止符 */
    return s;
}
```

### 扩容&空间预分配

大多数追加操作中，都会调用到扩容，其中根据新长度的大小，扩容大小也不同，一般为新长度的两倍，在大小为1Mb后，每次扩容变为增加1Mb，从而实现内存预分配内存

```c
#define SDS_MAX_PREALLOC (1024*1024)

if (newlen < SDS_MAX_PREALLOC)
    newlen *= 2;
else
    newlen += SDS_MAX_PREALLOC;
```

完整代码如下

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);  /* sds剩余可用 即s.alloc - s.len */
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* 有足够空间则返回 */
    if (avail >= addlen) return s;

    len = sdslen(s); /* 字符串长度 */
    sh = (char*)s-sdsHdrSize(oldtype);  /* sdshdr指针 */
    newlen = (len+addlen);  /* 新长度 */
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen); /* 新长度合适的sds类型 */

    /* 扩容时一般不为SDS_TYPE_5 */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type); /* 新sdshdr大小 */
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* 分配内存，拷贝原字符串，释放原sds内存，并设置好flags以及len */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}
```

### 缩容&惰性空间释放

当某些操作后，减少了原有字符串大小，sds并不会立即重新释放空间，重新分配内存，而是继续保留那么多空间，说不定下次就用上了。例如`sdsclear`操作

```c
// 将len字段设置为0，但内存空间不释放。
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}
```

当然必要的时候也可以调用`sdsRemoveFreeSpace`，主动缩容

```c
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    size_t avail = sdsavail(s);
    sh = (char*)s-oldhdrlen;

    /* 可用空间为0，说明没必要缩容 */
    if (avail == 0) return s;

    /* 通过比较sdshdrd大小*/
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);

    /* If the type is the same, or at least a large enough type is still
     * required, we just realloc(), letting the allocator to do the copy
     * only if really needed. Otherwise if the change is huge, we manually
     * reallocate the string to use the different header type. */
    if (oldtype==type || type > SDS_TYPE_8) {
        newsh = s_realloc(sh, oldhdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else { /* 空余空间较大，直接重新分配内存 */
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```

## 小结

sds相比于C语言字符串，具有以下几个显著的特点：

1. 结构中存储字符串长度，获取字符长度只需要常量时间复杂度，而原生字符串则需要遍历数组。
2. 二进制安全，可以存储字节数据，因为存储字符串长度，不会提前遇到\0字符而终止
3. 杜绝缓冲区溢出，C语言字符串本身不记录数组长度，增加操作时，分配内存不足时容易造成缓冲区溢出，而sds因为存在alloc，会在修改时，检查空间大小是否满足。
4. 内存预分配以及惰性删除，减少内存重新分配次数
5. 兼容C标准库中的部分字符串函数



## 参考

- [Redis sds](https://github.com/antirez/sds)
- [Redis设计与实现](https://redisbook.readthedocs.io/en/latest/internal-datastruct/sds.html)
- [Redis内部数据结构详解(2)——sds](http://zhangtielei.com/posts/blog-redis-sds.html)
