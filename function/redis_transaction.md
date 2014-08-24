#redis事务

事务是一个数据库必备的元素，对于redis也不例外，对于一个传统的关系型数据库来说，数据库事务满足ACID四个特性：

- A代表原子性：一个事务（transaction）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。
- C代表一致性：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中的数据应满足完整性约束
- I代表隔离性：多个事务并发执行时，一个事务的执行不应影响其他事务的执行
- D代表持久性：已被提交的事务对数据库的修改应该永久保存在数据库中

然而，对于redis来说，只满足其中的：

一致性和隔离性两个特性，其他特性是不支持的。

关于redis对ACID四个特性暂时先说这么多，在本文后面会详细说明。在详述之前，我们先来了解redis事务具体的使用和实现，这样我们接下来讨论ACID时才能更好的理解。

#redis事务的使用

redis事务主要包括MULTI、EXEC、DISCARD和WATCH命令

###MULTI命令

MULTI命令用来开启一个事务，当MULTI执行之后，客户端可以继续向服务器发送多条命令，这些命令会缓存在队列里面，只有当执行EXEC命令时，这些命令才会执行。

而DISCARD命令可以清空事务队列，放弃执行事务。

一个使用MULTI和EXEC执行事务的例子：

	> MULTI
	OK

	> INCR foo
	QUEUED

	> INCR bar
	QUEUED

	> EXEC
	1) (integer) 1
	2) (integer) 1

###redis事务产生错误

使用redis事务可能会产生错误，主要分为两大类：

- 事务在执行EXEC之前，入队的命令可能出错。
- 命令可能在 EXEC 调用之后失败

redis在2.6.5之后，如果发现事务在执行EXEC之前出现错误，那么会放弃这个事务。

在EXEC命令之后产生的错误，会被忽略，其他正确的命令会被继续执行。

###redis事务不支持回滚

如果你有使用关系式数据库的经验， 那么 “Redis 在事务失败时不进行回滚，而是继续执行余下的命令”这种做法可能会让你觉得有点奇怪。

以下是这种做法的优点：

- Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。
- 因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。
有种观点认为 Redis 处理事务的做法会产生 bug ， 然而需要注意的是， 在通常情况下， 回滚并不能解决编程错误带来的问题。 举个例子， 如果你本来想通过 INCR 命令将键的值加上 1 ， 却不小心加上了 2 ， 又或者对错误类型的键执行了 INCR ， 回滚是没有办法处理这些情况的。

鉴于没有任何机制能避免程序员自己造成的错误， 并且这类错误通常不会在生产环境中出现， 所以 Redis 选择了更简单、更快速的无回滚方式来处理事务。

###关于WATCH命令

WATCH命令可以添加监控的键，如果这些监控的键没有被其他客户端修改，那么事务可以顺利执行，如果被修改了，那么事务就不能执行。

##redis事务的实现

在了解了redis事务的使用之后，我们再来看看redis事务的实现，主要是对上面说的MULTI、EXEC、DISCARD和WATCH命令的源码的分析。

###MULTI命令实现

	void multiCommand(redisClient *c) {
    	if (c->flags & REDIS_MULTI) {
    	    addReplyError(c,"MULTI calls can not be nested");
    	    return;
   	 	}
    	c->flags |= REDIS_MULTI;
    	addReply(c,shared.ok);
	}

从上面的源码可以看出，MULTI只能执行一次，而且就做一件事，把客户端的标志打上REDIS_MULTI。

###EXEC命令实现

	void execCommand(redisClient *c) {
    	int j;
    	robj **orig_argv;
    	int orig_argc;
    	struct redisCommand *orig_cmd;

    	if (!(c->flags & REDIS_MULTI)) {
        	addReplyError(c,"EXEC without MULTI");
        	return;
    	}

    	/* Check if we need to abort the EXEC if some WATCHed key was touched.
     	* A failed EXEC will return a multi bulk nil object. */
    	if (c->flags & REDIS_DIRTY_CAS) {
        	freeClientMultiState(c);
        	initClientMultiState(c);
        	c->flags &= ~(REDIS_MULTI|REDIS_DIRTY_CAS);
        	unwatchAllKeys(c);
        	addReply(c,shared.nullmultibulk);
        	goto handle_monitor;
    	}

    	/* Replicate a MULTI request now that we are sure the block is executed.
     	* This way we'll deliver the MULTI/..../EXEC block as a whole and
     	* both the AOF and the replication link will have the same consistency
     	* and atomicity guarantees. */
    	execCommandReplicateMulti(c);

    	/* Exec all the queued commands */
    	unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */
    	orig_argv = c->argv;
    	orig_argc = c->argc;
    	orig_cmd = c->cmd;
    	addReplyMultiBulkLen(c,c->mstate.count);
    	for (j = 0; j < c->mstate.count; j++) {
        	c->argc = c->mstate.commands[j].argc;
        	c->argv = c->mstate.commands[j].argv;
        	c->cmd = c->mstate.commands[j].cmd;
        	call(c,REDIS_CALL_FULL);

        	/* Commands may alter argc/argv, restore mstate. */
        	c->mstate.commands[j].argc = c->argc;
        	c->mstate.commands[j].argv = c->argv;
        	c->mstate.commands[j].cmd = c->cmd;
    	}
    	c->argv = orig_argv;
    	c->argc = orig_argc;
    	c->cmd = orig_cmd;
    	freeClientMultiState(c);
    	initClientMultiState(c);
    	c->flags &= ~(REDIS_MULTI|REDIS_DIRTY_CAS);
    	/* Make sure the EXEC command is always replicated / AOF, since we
     	* always send the MULTI command (we can't know beforehand if the
     	* next operations will contain at least a modification to the DB). */
    	server.dirty++;

	handle_monitor:
    	/* Send EXEC to clients waiting data from MONITOR. We do it here
     	* since the natural order of commands execution is actually:
     	* MUTLI, EXEC, ... commands inside transaction ...
     	* Instead EXEC is flagged as REDIS_CMD_SKIP_MONITOR in the command
     	* table, and we do it here with correct ordering. */
    	if (listLength(server.monitors) && !server.loading)
        	replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
	}

其主要步骤是：

- 检查是否已经是处在MULTI状态下，如果不是，直接返回
- 检查WATCH的key是否已经被其他客户端修改，如果是，放弃事务
- 执行事务命令队列里面的所有命令

执行命令队列里面的所有命令的代码如下：

    	for (j = 0; j < c->mstate.count; j++) {
        	c->argc = c->mstate.commands[j].argc;
        	c->argv = c->mstate.commands[j].argv;
        	c->cmd = c->mstate.commands[j].cmd;
        	call(c,REDIS_CALL_FULL);

        	/* Commands may alter argc/argv, restore mstate. */
        	c->mstate.commands[j].argc = c->argc;
        	c->mstate.commands[j].argv = c->argv;
        	c->mstate.commands[j].cmd = c->cmd;
    	}

遍历queue队列的所有命令，一条条的执行

###DISCARD命令实现

DICARD命令主要由下面两个函数完成：

	void discardCommand(redisClient *c) {
    	if (!(c->flags & REDIS_MULTI)) {
    	    addReplyError(c,"DISCARD without MULTI");
        	return;
    	}
    	discardTransaction(c);
    	addReply(c,shared.ok);
	}

	void discardTransaction(redisClient *c) {
    	freeClientMultiState(c);
    	initClientMultiState(c);
    	c->flags &= ~(REDIS_MULTI|REDIS_DIRTY_CAS);;
    	unwatchAllKeys(c);
	}

	void freeClientMultiState(redisClient *c) {
    	int j;

    	for (j = 0; j < c->mstate.count; j++) {
        	int i;
        	multiCmd *mc = c->mstate.commands+j;

        	for (i = 0; i < mc->argc; i++)
            	decrRefCount(mc->argv[i]);
        	zfree(mc->argv);
    	}
    	zfree(c->mstate.commands);
	}

函数调用的关系如下

      discardCommand
           |
           |
           |
    discardTransaction
           |
           |
           |
    freeClientMultiState

freeClientMultiState就是完成DISCARD命令主要功能的函数，把其命令队列里面的所有命令所在的存储空间都释放。

###WATCH命令

	void watchForKey(redisClient *c, robj *key) {
    	list *clients = NULL;
    	listIter li;
    	listNode *ln;
    	watchedKey *wk;

    	/* Check if we are already watching for this key */
    	listRewind(c->watched_keys,&li);
    	while((ln = listNext(&li))) {
        	wk = listNodeValue(ln);
        	if (wk->db == c->db && equalStringObjects(key,wk->key))
            	return; /* Key already watched */
    	}
    	/* This key is not already watched in this DB. Let's add it */
    	clients = dictFetchValue(c->db->watched_keys,key);
    	if (!clients) { 
        	clients = listCreate();
        	dictAdd(c->db->watched_keys,key,clients);
        	incrRefCount(key);
    	}
    	listAddNodeTail(clients,c);
    	/* Add the new key to the lits of keys watched by this client */
    	wk = zmalloc(sizeof(*wk));
    	wk->key = key;
    	wk->db = c->db;
    	incrRefCount(key);
    	listAddNodeTail(c->watched_keys,wk);
	}

这个函数是在WATCH命令执行后，主要完成两件事情：
- 把key添加到当前客户端c->watched_keys的链表中，
- 并且把当前客户端添加到c->db->watched_keys的字典里面去。

当db中有能修改键状态的命令（比如INCR等）执行时，会自动把watch这个key的客户端的事务设置为放弃状态：

	void touchWatchedKey(redisDb *db, robj *key) {
    	list *clients;
    	listIter li;
    	listNode *ln;

    	if (dictSize(db->watched_keys) == 0) return;
    	clients = dictFetchValue(db->watched_keys, key);
    	if (!clients) return;

    	/* Mark all the clients watching this key as REDIS_DIRTY_CAS */
    	/* Check if we are already watching for this key */
    	listRewind(clients,&li);
    	while((ln = listNext(&li))) {
        	redisClient *c = listNodeValue(ln);

        	c->flags |= REDIS_DIRTY_CAS;
    	}
	}

如上源码所示，当某个key被修改后，会遭到其字典中所在的hash链表，然后把链表中所有的客户端都设置成REDIS_DIRTY_CAS状态，即事务被DISCARD的状态。

##redis事务的ACID性质讨论

###A原子性

redis事务在正常执行的时候是不会被打断的，但是redis事务至少在以下两个方面不满足原子性的要求：

- redis事务不支持回滚
- redis进程如果被kill等杀掉了，那么redis事务会执行一个，不满足要么全部全部执行，要么一条都不执行的条件

###C一致性

一致性分下面几种情况来讨论：

首先，如果一个事务的指令全部被执行，那么数据库的状态是满足数据库完整性约束的

其次，如果一个事务中有的指令有错误，那么数据库的状态是满足数据完整性约束的

最后，如果事务运行到某条指令时，进程被kill掉了，那么要分下面几种情况讨论：

- 如果当前redis采用的是内存模式，那么重启之后redis数据库是空的，那么满足一致性条件
- 如果当前采用RDB模式存储的，在执行事务时，Redis 不会中断事务去执行保存 RDB 的工作，只有在事务执行之后，保存 RDB 的工作才有可能开始。所以当 RDB 模式下的 Redis 服务器进程在事
务中途被杀死时，事务内执行的命令，不管成功了多少，都不会被保存到 RDB 文件里。
恢复数据库需要使用现有的 RDB 文件，而这个 RDB 文件的数据保存的是最近一次的数
据库快照（snapshot），所以它的数据可能不是最新的，但只要 RDB 文件本身没有因为
其他问题而出错，那么还原后的数据库就是一致的

- 如果当前采用的是AOF存储的，那么可能事务的内容还未写入到AOF文件，那么此时肯定是满足一致性的，如果事务的内容有部分写入到AOF文件中，那么需要用工具把AOF中事务执行部分成功的指令移除，这时，移除之后的AOF文件也是满足一致性的

所以，redis事务满足一致性约束

###I隔离性

Redis 是单进程程序，并且它保证在执行事务时，不会对事务进行中断，事务可以运行直到执
行完所有事务队列中的命令为止。因此，Redis 的事务是总是带有隔离性的。

###D持久性

根据在一致性的讨论知道，redis事务不满足持久性。
     


