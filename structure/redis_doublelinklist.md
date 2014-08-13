###redis双链表

在看redis双链表之前，我们先来看看平时用的双链表的结构体是怎么定义的:

    struct test
    {
        char *data;
        struct test *prev; 
        struct test *next;
    };

上面那种定义有一个缺陷，即每种结构体需要定义自己的双链表相关数据和操作，例如，我的程序需要增加一个新功能，用到了`struct test2`，并且希望test2也能是一个双链表，那么则需要这样定义：

    struct test2
    {
         char *data;
         struct test2 *prev;
         struct test2 *next;
    };

这样，每次我需要一个新的双链表时，都得在相应的结构体定义prev和next指针，并且，还都得实现一遍关于插入删除等操作，这种方式显然不利于代码重用。

redis用void *指针解决了上面的问题，首先来看看redis中的双链表的定义结构体定义：

    typedef struct listNode {
        struct listNode *prev;
        struct listNode *next;
        void *value;
    } listNode;

    typedef struct listIter {
        listNode *next;
        int direction;
    } listIter;

    typedef struct list {
        listNode *head;
        listNode *tail;
        void *(*dup)(void *ptr);
        void (*free)(void *ptr);
        int (*match)(void *ptr, void *key);
        unsigned long len;
    } list;

把和类型相关的数据用void *value来表示，这样就可以实现一个和类型无关的双链表。

在了解其链表定义之后，我们来看看redis双链表是怎么实现链表的一些基本功能，像创建链表、插入一个节点、删除一个节点、查找一个节点的。

**创建双链表**

    list *listCreate(void)
    {
        struct list *list;

        if ((list = zmalloc(sizeof(*list))) == NULL)
            return NULL;
        list->head = list->tail = NULL;
        list->len = 0;
        list->dup = NULL;
        list->free = NULL;
        list->match = NULL;
        return list;
    }

创建链表的主要过程是分配list结构体的内存空间，然后对里面的一些成员初始化，如果链表长度len初始化为0等等。

**插入一个节点**

    list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
        listNode *node;

        if ((node = zmalloc(sizeof(*node))) == NULL)
            return NULL;
        node->value = value;
        if (after) {
            node->prev = old_node;
            node->next = old_node->next;
            if (list->tail == old_node) {
                list->tail = node;
            }
        } else {
            node->next = old_node;
            node->prev = old_node->prev;
            if (list->head == old_node) {
                list->head = node;
            }
        }
        if (node->prev != NULL) {
            node->prev->next = node;
        }
        if (node->next != NULL) {
            node->next->prev = node;
        }
        list->len++;
        return list;
    }

插入一个节点的过程如下：

- 为新节点分配存储空间，并把node->value指向类型相关的value
- 如果after为1，则把node插入到old_node之后，如果after为0，则把node插入到old_node之前。并且需要考虑新插入节点是头节点和尾节点的情况
- 双链表的长度+1

**删除一个节点**

    void listDelNode(list *list, listNode *node)
    {
        if (node->prev)
            node->prev->next = node->next;
        else
            list->head = node->next;
        if (node->next)
            node->next->prev = node->prev;
        else
            list->tail = node->prev;
        if (list->free) list->free(node->value);
        zfree(node);
        list->len--;
    }

删除一个节点的操作如下：

- 更改节点前后节点的指针指向
- 释放value指向的内存空间，释放节点本身指向的内存空间
- 双链表长度-1

**查找一个节点**

在介绍查找节点之前，需要先介绍双链表的迭代器的实现，前面已经给出了迭代器的定义

    typedef struct listIter {
        listNode *next;
        int direction;
    } listIter;

双链表中和迭代器相关的操作有如下几个：

    /* Returns a list iterator 'iter'. After the initialization every
     * call to listNext() will return the next element of the list.
     *
     * This function can't fail. */
    listIter *listGetIterator(list *list, int direction)
    {
        listIter *iter;
    
        if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
        if (direction == AL_START_HEAD)
            iter->next = list->head;
        else
            iter->next = list->tail;
        iter->direction = direction;
        return iter;
    }

    /* Release the iterator memory */
    void listReleaseIterator(listIter *iter) {
        zfree(iter);
    }

    /* Create an iterator in the list private iterator structure */
    void listRewind(list *list, listIter *li) {
        li->next = list->head;
        li->direction = AL_START_HEAD;
    }

    void listRewindTail(list *list, listIter *li) {
        li->next = list->tail;
        li->direction = AL_START_TAIL;
    }

    /* Return the next element of an iterator.
     * It's valid to remove the currently returned element using
     * listDelNode(), but not to remove other elements.
     *
     * The function returns a pointer to the next element of the list,
     * or NULL if there are no more elements, so the classical usage patter
     * is:
     *
     * iter = listGetIterator(list,<direction>);
     * while ((node = listNext(iter)) != NULL) {
     *     doSomethingWith(listNodeValue(node));
     * }
     *
     * */
    listNode *listNext(listIter *iter)
    {
        listNode *current = iter->next;

        if (current != NULL) {
            if (iter->direction == AL_START_HEAD)
                iter->next = current->next;
            else
                iter->next = current->prev;
        }
        return current;
    }

listGetIterator函数是生成一个迭代器，如果direction == AL_START_HEAD，表示从前向后遍历，否则，则表示从后向前遍历

listReleaseIterator则是释放迭代器所在内存空间

listRewind表示把迭代器初始化为从前向后遍历

listRewindTail表示把迭代器初始化从后向前遍历

listNext则根据迭代器的类型，如果是AL_START_HEAD方向的，那么next = current->next;否则next = current->prev

了解了迭代器之后，我们就可以去看看双链表的查找节点是怎么实现的了

    /* Search the list for a node matching a given key.
     * The match is performed using the 'match' method
     * set with listSetMatchMethod(). If no 'match' method
     * is set, the 'value' pointer of every node is directly
     * compared with the 'key' pointer.
     *
     * On success the first matching node pointer is returned
     * (search starts from head). If no matching node exists
     * NULL is returned. */
    listNode *listSearchKey(list *list, void *key)
    {
        listIter *iter;
        listNode *node;

        iter = listGetIterator(list, AL_START_HEAD);
        while((node = listNext(iter)) != NULL) {
            if (list->match) {
                if (list->match(node->value, key)) {
                    listReleaseIterator(iter);
                    return node;
                }
            } else {
                if (key == node->value) {
                    listReleaseIterator(iter);
                    return node;
                }
            }
       } 
       listReleaseIterator(iter);
       return NULL;
    }

- 首先，获得一个从前向后遍历的迭代器
- 依次遍历整个链表，匹配node中value是否与提供的key匹配，如果匹配返回node，如果所有的都不匹配，返回NULL



