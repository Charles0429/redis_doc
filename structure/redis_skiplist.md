###基本的跳跃表

一个对跳跃表讲解比较详细的博文，大家可以去看看[浅析SkipList跳跃表原理及代码实现](http://blog.csdn.net/ict2014/article/details/17394259)

###redis中的跳跃表

redis中的跳跃表和普通跳跃表有几点不一样的地方：

- 节点的值允许相同
- 每个节点还有一个backward指向前一个元素的指针，这是为了从反方向来迭代整个跳跃表

结合上面博文，我们从下面几个方面讨论redis的跳跃表：

- 重要数据结构的定义
- 初始化表
- 查找
- 插入
- 删除
- 释放表

####重要数据结构的定义

	typedef struct zskiplistNode {
    	robj *obj;
   	 	double score;
    	struct zskiplistNode *backward;
    	struct zskiplistLevel {
        	struct zskiplistNode *forward;
        	unsigned int span;
    	} level[];
	} zskiplistNode;

	typedef struct zskiplist {
    	struct zskiplistNode *header, *tail;
    	unsigned long length;
    	int level;
	} zskiplist;

redis跳跃表中每个节点由obj(相当于key),score表示值，以及level数组，里面有forward指向后续节点；backward指向前一个节点


####初始化表

	zskiplist *zslCreate(void) {
    	int j;
    	zskiplist *zsl;

    	zsl = zmalloc(sizeof(*zsl));
    	zsl->level = 1;
    	zsl->length = 0;
    	zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    	for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        	zsl->header->level[j].forward = NULL;
        	zsl->header->level[j].span = 0;
    	}
    	zsl->header->backward = NULL;
    	zsl->tail = NULL;
    	return zsl;
	}

初始化的过程包括分配跳跃表的内存空间，把所有forward指针设置为NULL等初始化操作。

####查找

	unsigned long zslGetRank(zskiplist *zsl, double score, robj *o) {
    	zskiplistNode *x;
    	unsigned long rank = 0;
    	int i;

    	x = zsl->header;
    	for (i = zsl->level-1; i >= 0; i--) {
        	while (x->level[i].forward &&
            	(x->level[i].forward->score < score ||
                	(x->level[i].forward->score == score &&
                	compareStringObjects(x->level[i].forward->obj,o) <= 0))) {
         	   		rank += x->level[i].span;
            		x = x->level[i].forward;
        		}

        		/* x might be equal to zsl->header, so test if obj is non-NULL */
        		if (x->obj && equalStringObjects(x->obj,o)) {
            		return rank;
       	 	}
    	}
    	return 0;
	}

上面的函数严格来讲不是查找，但是包含了一个跳跃表查找时所必须要做的操作，所以，就以这个函数来说明查找的过程。

查找的基本思想就是：

在每一层都找到最后一个比要查找元素小的元素，假设用指针p表示这个元素，那么要查找在元素就一定在[p,p->forward[i]]中，其中i表示当前的层。

然后，在把i减1，继续往下找，当到最后一层时，如果p->forward[0]的值和要查找的值相同，那么返回找到的指针，否则返回未找到。具体的这个过程可以结合上面博文分析。

####插入

插入主要由下面的函数完成：

	zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    	zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    	unsigned int rank[ZSKIPLIST_MAXLEVEL];
    	int i, level;

    	redisAssert(!isnan(score));
    	x = zsl->header;
    	for (i = zsl->level-1; i >= 0; i--) {
        	/* store rank that is crossed to reach the insert position */
        	rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        	while (x->level[i].forward &&
            	(x->level[i].forward->score < score ||
            	    (x->level[i].forward->score == score &&
            	    compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            	rank[i] += x->level[i].span;
            	x = x->level[i].forward;
        	}
        	update[i] = x;
    	}
    	/* we assume the key is not already inside, since we allow duplicated
     	* scores, and the re-insertion of score and redis object should never
     	* happpen since the caller of zslInsert() should test in the hash table
     	* if the element is already inside or not. */
    	level = zslRandomLevel();
    	if (level > zsl->level) {
        	for (i = zsl->level; i < level; i++) {
        	    rank[i] = 0;
        	    update[i] = zsl->header;
        	    update[i]->level[i].span = zsl->length;
        	}
        	zsl->level = level;
    	}
    	x = zslCreateNode(level,score,obj);
    	for (i = 0; i < level; i++) {
        	x->level[i].forward = update[i]->level[i].forward;
        	update[i]->level[i].forward = x;

        	/* update span covered by update[i] as x is inserted here */
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

从上面代码可以看出，跳跃表的插入一个节点的主要步骤是：

- 和插入过程类似，获得要插入的节点在每一层的前驱指针
- 随机生成要插入节点的level
- 更新前驱指针的forward指针，使其指向新插入的节点，以及要插入节点的forward指针
- 更新backward指针（这个是在跳跃表最底层用到）
- 更新跳跃表长度

####删除

redis跳跃表的删除主要由下面两个函数完成：

	/* Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank */
	void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
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
    	while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
    	    zsl->level--;
    	zsl->length--;
	}

	/* Delete an element with matching score/object from the skiplist. */
	int zslDelete(zskiplist *zsl, double score, robj *obj) {
    	zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    	int i;

    	x = zsl->header;
    	for (i = zsl->level-1; i >= 0; i--) {
        	while (x->level[i].forward &&
        	    (x->level[i].forward->score < score ||
        	        (x->level[i].forward->score == score &&
        	        compareStringObjects(x->level[i].forward->obj,obj) < 0)))
        	    x = x->level[i].forward;
        	update[i] = x;
    	}
    	/* We may have multiple elements with the same score, what we need
     	* is to find the element with both the right score and object. */
    	x = x->level[0].forward;
    	if (x && score == x->score && equalStringObjects(x->obj,obj)) {
        	zslDeleteNode(zsl, x, update);
        	zslFreeNode(x);
        	return 1;
    	} else {
        	return 0; /* not found */
    	}
    	return 0; /* not found */
	}

从上面代码可以看出，删除过程如下：

- 首先找到要删除节点的在所有层的前驱指针
- 然后对要删除节点所在的那些层，更新其前驱指针的forward指针，指向要删除节点的后驱指针
- 更新要删除节点的后驱指针的backward指针域
- 可能会更新跳跃表的总层数
- 把跳跃表的长度减1

####释放表

    void zslFree(zskiplist *zsl) {
        zskiplistNode *node = zsl->header->level[0].forward, *next;

        zfree(zsl->header);
        while(node) {
            next = node->level[0].forward;
        	zslFreeNode(node);
        	node = next;
    	}
    	zfree(zsl);
	}

释放表的操作很简单，因为跳跃表第0层包含了所有节点，和普通的链表一样，所以，按照普通链表的方式来释放空间即可。