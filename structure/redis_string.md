###简介

redis字符串有下面几个特点：

- O(1)时间内计算字符串的长度
- 自己管理内存分配，字符串使用者不需要关系内存空间是否不够等细节问题
- 与普通字符串一样的定义，redis的字符串使用时，真正用到的还是char *类型，使得redis字符串可以出现在普通字符串出现的地方

###实现

####redis字符串结构体用到的技巧

先看redis string的结构体

	typedef char *sds;

	struct sdshdr {
    	int len;
    	int free;
    	char buf[];
	};

初看上面的定义，如果对C99标准的Zero length array不熟悉的话，可能会对定义中的`char buf[]`产生疑问，这个没有长度的字符串代表了什么？

为了说明清楚上面的问题，我们先来看在C90标准中一个结构体的使用技巧：

    struct test
    {
        int len;
        int free;
        char buf[1];
    };

在上面的结构体基础上，我们可以做如下的内存分配：

    struct test * p = malloc(sizeof(test) + (16 - 1));

然后，你可以这样操作数据：

    p->buf[12] = '1';

就像buf指向了一个数组一样，即`sizeof(test) + (16-1)`后面的16-1就是p->buf指向的空间。

那么，为什么需要这种技巧呢？

首先，如果我们用一般的实现，我们可能会把`char buf[1]`的定义换成`char *buf`

    struct test
    {
        int len;
        int free;
        char *buf;
    };

这样，我们为test分配内存空间和`char *buf`分配内存空间需要两次malloc操作，而用`char buf[1]`定义只需要一次malloc

其次，一次malloc分配的内存在实际内存中是相邻的，其cache命中率会比两次的malloc的可能要高。

基于上面两个好处，C99就从语言本身支持zero length array来支持上面的特性，即redis字符串结构体定义的那样

    struct sdshdr {
    	int len;
    	int free;
    	char buf[];
	};

这样方法比`char buf[1]`的好处是省去一个额外的字节。

####redis字符串结构体各项的含义

    struct sdshdr{
        int len;
        int free;
        char buf[];
     };

len表示字符串的实际长度（不包括'\0'），free表示当前剩余的内存空间，buf则是存储字符串的起始地址。由于'\0'的存在，所以，字符串buf所指向的实际内存空间为 len + free + 1。

####redis字符串各项操作的实现

**计算字符串长度**

由于结构体中已经包含了长度信息，所以直接返回len即可。

**redis字符串连接**

这个功能相当于C里面的strcat函数

    sds sdscatlen(sds s, const void *t, size_t len) {
        struct sdshdr *sh;
        size_t curlen = sdslen(s);

        s = sdsMakeRoomFor(s,len);
        if (s == NULL) return NULL;
        sh = (void*) (s-(sizeof(struct sdshdr)));
        memcpy(s+curlen, t, len);
        sh->len = curlen+len;
        sh->free = sh->free-len;
        s[curlen+len] = '\0';
        return s;
    }

其中sdsMakeroomFor(s, len)是为字符串s分配足够的空间，来容纳后面添加len个字符，sdsMakeRoomFor的策略如下：

    sds sdsMakeRoomFor(sds s, size_t addlen) {
        struct sdshdr *sh, *newsh;
        size_t free = sdsavail(s);
        size_t len, newlen;

        if (free >= addlen) return s;
        len = sdslen(s);
        sh = (void*) (s-(sizeof(struct sdshdr)));
        newlen = (len+addlen);
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;
        newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
        if (newsh == NULL) return NULL;

        newsh->free = newlen - len;
        return newsh->buf;
    }

- 如果当前字符串s的free>=len,那么表明剩余空间足够容纳len个字符，直接返回即可
- 如果当前字符串s的free<len,那么根据len+addlen的情况，来决定新增的内存空间大小，如果len+addlen已经大于或等于SDS_MAX_PREALLOC了，那么就多增加SDS_MAX_PREALLOC，否则就新增len+addlen空间。

sdscatlen函数的后面的语句就是把字符串t的len个字节用memcpy到当前字符串s的后面，然后更新s的free和len信息

**redis字符串复制**

这个功能相当于C里面的strcpy函数

	sds sdscpylen(sds s, const char *t, size_t len) {
    	struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
    	size_t totlen = sh->free+sh->len;

    	if (totlen < len) {
        	s = sdsMakeRoomFor(s,len-sh->len);
        	if (s == NULL) return NULL;
        	sh = (void*) (s-(sizeof(struct sdshdr)));
        	totlen = sh->free+sh->len;
    	}
    	memcpy(s, t, len);
    	s[len] = '\0';
    	sh->len = len;
    	sh->free = totlen-len;
    	return s;
	}

这个函数和前面的sdscatlen函数基本上是类似的，也是先检查是否需要分配空间，然后拷贝字符串，最后更新free和len信息。

