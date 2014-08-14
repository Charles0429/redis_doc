###redis整数集合

redis整数集合，即intset，是当集合中所有元素都是整数的时候，才能使用的一种数据结构，主要有下面几个特点：

- 比使用字符串保存数据更省内存，但是可能会更耗CPU
- 有三种编码方式，当所有的整数可以用int16_t来表示的时候，就用int16_t来表示所有整数；当所有整数的范围都处于int16_t和int32_t之间的时候，就会用int32_t来表示；当所有整数的范围都处于int32_t到int64_t的时候，就会用int64_t来表示

###具体实现

####结构体定义

	typedef struct intset {
    	uint32_t encoding;
    	uint32_t length;
    	int8_t contents[];
	} intset;
	#define INTSET_ENC_INT16 (sizeof(int16_t))
	#define INTSET_ENC_INT32 (sizeof(int32_t))
	#define INTSET_ENC_INT64 (sizeof(int64_t))

首先encoding就是上面说的三种长度的整数集合，length是只当前保存的整数的个数，而contents则是用来保存具体的数据内容。

redis整数集合涉及到的操作主要是整数集合的创建、查找元素、添加元素、删除元素等等

####整数集合的创建

整数集合的创建由下面的函数完成工作

	intset *intsetNew(void) {
    	intset *is = zmalloc(sizeof(intset));
    	is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    	is->length = 0;
    	return is;
	}

这个函数就要是完成整数集合结构体的内存分配，以及设置默认的encoding为int16_t类型的。

####查找元素

	static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    	int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    	int64_t cur = -1;

    	/* The value can never be found when the set is empty */
    	if (intrev32ifbe(is->length) == 0) {
        	if (pos) *pos = 0;
        	return 0;
    	} else {
        	/* Check for the case where we know we cannot find the value,
         	* but do know the insert position. */
        	if (value > _intsetGet(is,intrev32ifbe(is->length)-1)) {
            	if (pos) *pos = intrev32ifbe(is->length);
            	return 0;
        	} else if (value < _intsetGet(is,0)) {
            	if (pos) *pos = 0;
            	return 0;
        	}
    	}

    	while(max >= min) {
        	mid = (min+max)/2;
        	cur = _intsetGet(is,mid);
        	if (value > cur) {
            	min = mid+1;
        	} else if (value < cur) {
            	max = mid-1;
        	} else {
            	break;
        	}
    	}

    	if (value == cur) {
        	if (pos) *pos = mid;
        	return 1;
    	} else {
        	if (pos) *pos = min;
        	return 0;
    	}
	}

查找过程如上代码所示，主要是下面几个步骤：

- 如果表长为0，那么一定找不到，返回可插入的位置*pos=0
- 如果要查找的元素小于第一个元素或者小于最后一个元素，那么也可以快速的确定这个元素不在集合中，而且其插入位置也可以知道(整数集合是有序的)
- 如果不满足上面两点要求，那么使用二分查找来确定元素是否存在。

####添加元素

添加元素的操作用到了主要用到了下面几个函数

    intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
        uint8_t valenc = _intsetValueEncoding(value);
        uint32_t pos;
        if (success) *success = 1;
    
        /* Upgrade encoding if necessary. If we need to upgrade, we know that
         * this value should be either appended (if > 0) or prepended (if < 0),
         * because it lies outside the range of existing values. */
        if (valenc > intrev32ifbe(is->encoding)) {
            /* This always succeeds, so we don't need to curry *success. */
            return intsetUpgradeAndAdd(is,value);
        } else {
            /* Abort if the value is already present in the set.
             * This call will populate "pos" with the right position to insert
             * the value when it cannot be found. */
            if (intsetSearch(is,value,&pos)) {
                if (success) *success = 0;
                return is;
            }
    
            is = intsetResize(is,intrev32ifbe(is->length)+1);
            if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
        }
    
        _intsetSet(is,pos,value);
        is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
        return is;
    }


    static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
        uint8_t curenc = intrev32ifbe(is->encoding);
        uint8_t newenc = _intsetValueEncoding(value);
        int length = intrev32ifbe(is->length);
        int prepend = value < 0 ? 1 : 0;
    
        /* First set new encoding and resize */
        is->encoding = intrev32ifbe(newenc);
        is = intsetResize(is,intrev32ifbe(is->length)+1);
    
        /* Upgrade back-to-front so we don't overwrite values.
         * Note that the "prepend" variable is used to make sure we have an empty
         * space at either the beginning or the end of the intset. */
        while(length--)
            _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));
    
        /* Set the value at the beginning or the end. */
        if (prepend)
            _intsetSet(is,0,value);
        else
            _intsetSet(is,intrev32ifbe(is->length),value);
        is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
        return is;
    }

所以，插入操作主要是下面几个步骤：

- 当新插入的元素用当前整数集合的编码(encoding)表示不了的时候，就需要把当前集合的encoding升级到要表示要插入集合元素的encoding
- 否则的话，就先查找要插入的整数是否已经存在于整数集合中，如果已经存在了，就不需要插入了
- 如果不存在，那么就和数组中插入数据一样，需要把数据从后面到要插入的位置的数据都向后挪动一个单位，然后插入数据

上面步骤中的整数集合的编码提升是由intsetUpgradeAndAdd完成的，主要的工作是

- 重新分配足够大的空间，来准备整数提升
- 从后往前开始把整数挪动到新空间适合的位置，从后往前挪动的原因是防止数据覆盖

####删除元素

	/* Delete integer from intset */
	intset *intsetRemove(intset *is, int64_t value, int *success) {
    	uint8_t valenc = _intsetValueEncoding(value);
    	uint32_t pos;
    	if (success) *success = 0;

    	if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        	uint32_t len = intrev32ifbe(is->length);

        	/* We know we can delete */
        	if (success) *success = 1;

        	/* Overwrite value with tail and update length */
        	if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        	is = intsetResize(is,len-1);
        	is->length = intrev32ifbe(len-1);
    	}
    	return is;
	}

	static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    	void *src, *dst;
    	uint32_t bytes = intrev32ifbe(is->length)-from;
    	uint32_t encoding = intrev32ifbe(is->encoding);

    	if (encoding == INTSET_ENC_INT64) {
        	src = (int64_t*)is->contents+from;
        	dst = (int64_t*)is->contents+to;
        	bytes *= sizeof(int64_t);
    	} else if (encoding == INTSET_ENC_INT32) {
        	src = (int32_t*)is->contents+from;
        	dst = (int32_t*)is->contents+to;
        	bytes *= sizeof(int32_t);
    	} else {
        	src = (int16_t*)is->contents+from;
        	dst = (int16_t*)is->contents+to;
        	bytes *= sizeof(int16_t);
    	}
    	memmove(dst,src,bytes);
	}

首先，intsetMoveTail函数是把从下标from开始的数据复制到以to为下标开始的地方，因为源地址和目标地址可能有重叠，所以用的是memmove函数，能正确处理有内存重叠的情况。

我们知道，在数组中，删除一个元素的就是把其后面的元素一次往前挪动一个数据大小的空间位置即可，所以，在intsetRemove中，使用intsetMoveTail，把pos+1开始的下标的数据，全部往前挪动一个单位到pos开始的位置，这样就完成了对pos位置数据的删除。


