###redis字典

redis字典字典是基于hash表实现的。

一般的hash表要考虑下面几个因素：

- hash算法，具体从key映射到hash表中具体下标的算法
- 解决冲突的办法
- 当hash表装填引子到达某个比例是，需要进行扩张

下面就从上面说的几个因素来看redis的字典

###hash算法


    /* Thomas Wang's 32 bit Mix Function */
    unsigned int dictIntHashFunction(unsigned int key)
    {
        key += ~(key << 15);
        key ^=  (key >> 10);
        key +=  (key << 3);
        key ^=  (key >> 6);
        key += ~(key << 11);
        key ^=  (key >> 16);
        return key;
    }
    
    /* Identity hash function for integer keys */
    unsigned int dictIdentityHashFunction(unsigned int key)
    {
        return key;
    }
    
    static uint32_t dict_hash_function_seed = 5381;
    
    void dictSetHashFunctionSeed(uint32_t seed) {
        dict_hash_function_seed = seed;
    }
    
    uint32_t dictGetHashFunctionSeed(void) {
        return dict_hash_function_seed;
    }
    
    /* MurmurHash2, by Austin Appleby
     * Note - This code makes a few assumptions about how your machine behaves -
     * 1. We can read a 4-byte value from any address without crashing
     * 2. sizeof(int) == 4
     *
     * And it has a few limitations -
     *
     * 1. It will not work incrementally.
     * 2. It will not produce the same results on little-endian and big-endian
     *    machines.
     */
    unsigned int dictGenHashFunction(const void *key, int len) {
        /* 'm' and 'r' are mixing constants generated offline.
         They're not really 'magic', they just happen to work well.  */
        uint32_t seed = dict_hash_function_seed;
        const uint32_t m = 0x5bd1e995;
        const int r = 24;
    
        /* Initialize the hash to a 'random' value */
        uint32_t h = seed ^ len;
    
        /* Mix 4 bytes at a time into the hash */
        const unsigned char *data = (const unsigned char *)key;
    
        while(len >= 4) {
            uint32_t k = *(uint32_t*)data;
    
            k *= m;
            k ^= k >> r;
            k *= m;
    
            h *= m;
            h ^= k;
    
            data += 4;
            len -= 4;
        }
    
        /* Handle the last few bytes of the input array  */
        switch(len) {
        case 3: h ^= data[2] << 16;
        case 2: h ^= data[1] << 8;
        case 1: h ^= data[0]; h *= m;
        };
    
        /* Do a few final mixes of the hash to ensure the last few
         * bytes are well-incorporated. */
        h ^= h >> 13;
        h *= m;
        h ^= h >> 15;
    
        return (unsigned int)h;
    }
    
    /* And a case insensitive hash function (based on djb hash) */
    unsigned int dictGenCaseHashFunction(const unsigned char *buf, int len) {
        unsigned int hash = (unsigned int)dict_hash_function_seed;
    
        while (len--)
            hash = ((hash << 5) + hash) + (tolower(*buf++)); /* hash * 33 + c */
        return hash;
    }

上面的就是redis字典中具体用到的hash算法，这些算法一般都是基于某篇论文里面提出的算法实现的。

###解决冲突的办法

由于redis字典采用的是链式hash表，所以解决冲突的办法就是，往链表插入冲突的元素即可。

###redis字典扩张

首先，我们需要了解装填因子的概念，即hash表中已经存储元素的个数/hash表的大小，当装填引子大于某个值时，一般就要进行扩张，否则，hash的查找效率会变低。

redis判断字典是否需要进行扩张的代码如下：

    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, ((d->ht[0].size > d->ht[0].used) ?
                                    d->ht[0].size : d->ht[0].used)*2);
    }

可以看出，当redis的装填因子 > 1 且 dict_can_resize为真的时候，就需要扩张了。
当redis的装填因子 > dict_force_resize_ratio(即5)时，不管dict_can_resize是否为真，都需要扩张

###redis字典具体实现

####redis字典定义

    typedef struct dictEntry { 
        void *key;
        union {
            void *val;
            uint64_t u64;
            int64_t s64;
        } v;
        struct dictEntry *next;
    } dictEntry;

    typedef struct dictType {
        unsigned int (*hashFunction)(const void *key);
        void *(*keyDup)(void *privdata, const void *key);
        void *(*valDup)(void *privdata, const void *obj);
        int (*keyCompare)(void *privdata, const void *key1, const void *key2);
        void (*keyDestructor)(void *privdata, void *key);
        void (*valDestructor)(void *privdata, void *obj);
    } dictType;

    /* This is our hash table structure. Every dictionary has two of this as we
     * implement incremental rehashing, for the old to the new table. */
    typedef struct dictht {
        dictEntry **table;
        unsigned long size;
        unsigned long sizemask;
        unsigned long used;
    } dictht;

    typedef struct dict {
        dictType *type;
        void *privdata;
        dictht ht[2];
        int rehashidx; /* rehashing not in progress if rehashidx == -1 */
        int iterators; /* number of iterators currently running */
    } dict;

首先，dictEntry结构体就是hash表的一个节点，而dictht则是hash表的定义，而一个字典有两个hash表，一个是ht[0]，一个是ht[1]，ht[1]平时不使用，只有当字典要扩张的时候，才会使用。

了解了字典的结构体定义之后，再来了解其中常用的操作，包括：字典的创建、添加一个元素、删除一个元素、查找一个元素、字典扩张过程等。

**字典的创建**

字典的创建由下面的函数完成的：

    dict *dictCreate(dictType *type,
        void *privDataPtr)
    {
        dict *d = zmalloc(sizeof(*d));

        _dictInit(d,type,privDataPtr);
        return d;
    }

具体的就是分配字典所需的内存空间，初始化字典内部的成员

**添加一个元素**

字典添加一个元素主要由下面两个函数和一个宏定义完成的：

	/* Add an element to the target hash table */
	int dictAdd(dict *d, void *key, void *val)
	{
    	dictEntry *entry = dictAddRaw(d,key);

    	if (!entry) return DICT_ERR;
    	dictSetVal(d, entry, val);
    	return DICT_OK;
	}

	dictEntry *dictAddRaw(dict *d, void *key)
	{
    	int index;
    	dictEntry *entry;
    	dictht *ht;

    	if (dictIsRehashing(d)) _dictRehashStep(d);

    	/* Get the index of the new element, or -1 if
     	* the element already exists. */
    	if ((index = _dictKeyIndex(d, key)) == -1) //定位在hash表的具体下标
        	return NULL;

    	/* Allocate the memory and store the new entry */
    	ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    	entry = zmalloc(sizeof(*entry));
    	entry->next = ht->table[index];
    	ht->table[index] = entry;
    	ht->used++;

    	/* Set the hash entry fields. */
    	dictSetKey(d, entry, key);
    	return entry;
	}

	#define dictSetVal(d, entry, _val_) do { \
    	if ((d)->type->valDup) \
    	    entry->v.val = (d)->type->valDup((d)->privdata, _val_); \
    	else \
        	entry->v.val = (_val_); \
	} while(0)

具体的添加过程如下：

- 调用dictAddRaw分配一个链表的节点空间
- 调用dictSetVal函数，把val指向的空间复制一份，并且将分配的节点的val指向这个复制出来的地址

**删除一个元素**

删除一个元素主要由下面三个函数实现的：

    /* Search and remove an element */
    static int dictGenericDelete(dict *d, const void *key, int nofree)
    {
        unsigned int h, idx;
        dictEntry *he, *prevHe;
        int table;
    
        if (d->ht[0].size == 0) return DICT_ERR; /* d->ht[0].table is NULL */
        if (dictIsRehashing(d)) _dictRehashStep(d);
        h = dictHashKey(d, key);
    
        for (table = 0; table <= 1; table++) {
            idx = h & d->ht[table].sizemask;
            he = d->ht[table].table[idx];
            prevHe = NULL;
            while(he) {
                if (dictCompareKeys(d, key, he->key)) {
                    /* Unlink the element from the list */
                    if (prevHe)
                        prevHe->next = he->next;
                    else
                        d->ht[table].table[idx] = he->next;
                    if (!nofree) {
                        dictFreeKey(d, he);
                        dictFreeVal(d, he);
                    }
                    zfree(he);
                    d->ht[table].used--;
                    return DICT_OK;
                }
                prevHe = he;
                he = he->next;
            }
            if (!dictIsRehashing(d)) break;
        }
        return DICT_ERR; /* not found */
    }
    
    int dictDelete(dict *ht, const void *key) {
        return dictGenericDelete(ht,key,0);
    }
    
    int dictDeleteNoFree(dict *ht, const void *key) {
        return dictGenericDelete(ht,key,1);
    }

先来看dictGenericDelete的流程：

- 首先调用dictHashKey获取key所在的hash表的下标
- 根据下标，得到该下标首元素的地址，然后开始遍历整个链表，比较是否有节点的key和函数参数中的key相同，如果有，则从链表中删除这个节点；否则返回DICT_ERR

在dictGenericDelete的基础上，dictDelete删除key对应的元素的同时，会把其所在空间也删除，而dictDeleteNoFree则只会在hash表中删除元素，并不会删除其所在空间。

**查找一个元素**

查找元素主要通过下面的函数实现的

    dictEntry *dictFind(dict *d, const void *key)
    {
        dictEntry *he;
        unsigned int h, idx, table;

        if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */
        if (dictIsRehashing(d)) _dictRehashStep(d);
        h = dictHashKey(d, key);
        for (table = 0; table <= 1; table++) {
            idx = h & d->ht[table].sizemask;
            he = d->ht[table].table[idx];
            while(he) {
                if (dictCompareKeys(d, key, he->key))
                    return he;
                he = he->next;
            }
            if (!dictIsRehashing(d)) return NULL;
        }
        return NULL;
    }

dictFind操作其实在上面的dictGenericDelete的已经有所体现了：

- 先调用dictHashKey，获取在hash表中的下标
- 遍历下标指向的链表，查找是否有和函数参数中key相同的元素

**字典扩张**

字典扩展分为下面几个过程,首先是字典扩张初始化的过程

**字典扩张初始过程**

    int dictExpand(dict *d, unsigned long size)
    {
        dictht n; /* the new hash table */
        unsigned long realsize = _dictNextPower(size);

        /* the size is invalid if it is smaller than the number of
         * elements already inside the hash table */
        if (dictIsRehashing(d) || d->ht[0].used > size)
            return DICT_ERR;

        /* Allocate the new hash table and initialize all pointers to NULL */
        n.size = realsize;
        n.sizemask = realsize-1;
        n.table = zcalloc(realsize*sizeof(dictEntry*));
    	n.used = 0;

    	/* Is this the first initialization? If so it's not really a rehashing
     	* we just set the first hash table so that it can accept keys. */
    	if (d->ht[0].table == NULL) {
        	d->ht[0] = n;
        	return DICT_OK;
    	}

    	/* Prepare a second hash table for incremental rehashing */
    	d->ht[1] = n;
    	d->rehashidx = 0;
    	return DICT_OK;
	}

字典扩张初始化的过程就是给ht[1]表来分配存储空间，为扩张作初始准备

**字典扩张具体过程**

	/* Performs N steps of incremental rehashing. Returns 1 if there are still
	 * keys to move from the old to the new hash table, otherwise 0 is returned.
	 * Note that a rehashing step consists in moving a bucket (that may have more
	 * thank one key as we use chaining) from the old to the new hash table. */
	int dictRehash(dict *d, int n) {
    	if (!dictIsRehashing(d)) return 0;

    	while(n--) {
    	    dictEntry *de, *nextde;

    	    /* Check if we already rehashed the whole table... */
    	    if (d->ht[0].used == 0) {
    	        zfree(d->ht[0].table);
    	        d->ht[0] = d->ht[1];
    	        _dictReset(&d->ht[1]);
    	        d->rehashidx = -1;
    	        return 0;
    	    }

    	    /* Note that rehashidx can't overflow as we are sure there are more
    	     * elements because ht[0].used != 0 */
    	    assert(d->ht[0].size > (unsigned)d->rehashidx);
    	    while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;
    	    de = d->ht[0].table[d->rehashidx];
    	    /* Move all the keys in this bucket from the old to the new hash HT */
        	while(de) {
            	unsigned int h;

            	nextde = de->next;
            	/* Get the index in the new hash table */
            	h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            	de->next = d->ht[1].table[h];
            	d->ht[1].table[h] = de;
            	d->ht[0].used--;
            	d->ht[1].used++;
            	de = nextde;
        	}
        	d->ht[0].table[d->rehashidx] = NULL;
        	d->rehashidx++;
    	}
    	return 1;
	}

字典具体的扩张过程是由上面dictRehash实现的，具体的操作就是把字典中ht[0]的元素移动到ht[1]中，也就是从ht[0]的下标0开始，一个个的把对应下标的链表中所有元素一个个添加到ht[1]中。当全部元素都迁移到ht[1]中之后，使得ht[0]=ht[1]。

备注：因为ht[1]的hash表的大小发生了变化，所以，需要用dictHashKey进行重新计算hash表的下标

因为redis是单线程的，如果把一次性的把ht[0]的元素全部移到ht[1]中，可能会造成redis长时间无法对客户服务，所以在redis中一般采用渐进式的扩张。

    static void _dictRehashStep(dict *d) {
        if (d->iterators == 0) dictRehash(d,1);
    }
也就是每一次只执行一个下标对应的链表中元素的移动。

目前redis中，每次执行一次添加、查找、删除操作，_dictRehashStep 都会被执行一次。

