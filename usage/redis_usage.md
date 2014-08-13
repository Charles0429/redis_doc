###基础命令

**key相关的命令**

- KEYS glob通配符
- DEL key1[key2..keyn]
- EXIST key
    
    	import redis
    	def common():
    	client = redis.StrictRedis();
    	print client.set('foo', 1)
    	print client.hset('hash_foo', 'name', 1)
    	print client.keys('*')
    	print client.type('foo')
    	print client.type('hash_foo')
    	print client.exists('foo')
    	print client.delete(('foo', 'hash_foo'))
    	print client.keys('*')
    
    	if __name__ == '__main__':
    	common()  

上面的程序对应的输出如下：
    
    True
    0
    ['foo', 'hash_foo']
    string
    hash
    True
    2
    []

###基础数据类型-字符串

字符串的常用命令有如下几个：

- set key value
- get key
- incr key 对value增加1
- decr key 对value减少1

用字符串结构来实现一个博客文章的基本功能。

首先，每次新建一篇文章时，文章的ID会自动加1，这个可以定义一个key-value来存自增的文章ID，通过incr来实现递增

第二，文章本身数据的保存

一遍文章可能包括作者，标题，生成日期等等，如果想要用字符串统一保存的话，则需要把文章的数据序列化后才能保存到字符串中

第三，文章访问量

和传统的关系数据库不同，如果把文章访问量放在上面的序列化字符串中，那么每次有个读者读了这篇文章，就得把新增的访问次数重新序列化成字符串并且上传到服务器，这样会造成许多不必要的数据上传。所以，我们需要为每篇文章设置一个key-value，保存其访问量

	import redis
	import json
	import time

	def init(client):
    	client.set('post-global-id', 0)

	def create_a_post(client, author, title, date):
    	dict = {}
    	dict['author'] = author
    	dict['title'] = title
    	dict['date'] = date

    	id = client.incr('post-global-id', 1)
    	content = json.dumps(dict)
    	client.set('post' + str(id), content)
    	return id

	def get_a_post(client, id):

    	content = client.get('post' + str(id))
    	dict = json.loads(content)
    	print id, dict['author'], dict['title'], dict['date']

	def main():
    	client = redis.StrictRedis()
    	init(client)
    	id = create_a_post(client, 'Charles', 'a blog', time.time())
    	get_a_post(client, id)

	if __name__ == '__main__':
    	main()

其他字符串的相关命令：

- incrby key number 对value增加member
- decrby key number 对value减少member
- append key content 对value后面追加content
- STRLEN key 获取key的长度
- MGET key[key..] 获取多个key的value
- MSET key value[key value...] 设置多个key-value键值对

###基础数据类型-散列

散列的一些常用的命令：

- hset key field value 设置一个key的field对应的value值
- hget key field 获取一个key的field对应的value值
- hmset key field value[field value...] 设置多个
- hmget key field[field...] 获取多个
- hgetall key 获取key的所有field的信息
- hexists key field　判断key中的field是否存在
- hsetnx key field value 如果key的field不存在则赋值，否则不做任何操作
- hincrby key field

前面对博客文章的作者，标题，时间等等字段采用序列化成字符串之后，再保存到服务器。获取的时候又需要反序列化，消耗了很多的时间，从散列的命令来看，它的key对应者多个field，这些field可以用来保存作者，标题等等信息，可以省去博文上传和下载过程中的序列化和反序列化过程。上面的代码就实现了这个过程：

	import redis
	import time

	def init(client):
    	client.set('post-global-id', 0)

	def create_a_post(client, author, title, date):
    	dict = {}
    	dict['author'] = author
    	dict['title'] = title
    	dict['date'] = date

    	id = client.incr('post-global-id', 1)
    	for k, v in dict.iteritems():
        	client.hset('post' + str(id), k, v)
    	return id

	def get_a_post(client, id):

    	dict = client.hgetall('post' + str(id))
    	print dict
    	print id,
    	for k,v in dict.iteritems():
        	print v,

	def main():
    	client = redis.StrictRedis()
    	init(client)
    	id = create_a_post(client, 'Charles', 'a blog', time.time())
    	get_a_post(client, id)

	if __name__ == '__main__':
    	main()

###基础数据类型-列表

在redis中，列表是排好序的，但其底层实现用的是双链表，所以，如果要查找一个在中间的元素，时间复杂度会是O(N)。不过，其查找位于两端的数据是比较快的。

列表基础命令：

- LPUSH key value[value...] 在列表左边插入数据
- RPUSH key value[value...] 在列表右边插入数据
- LPOP key 弹出列表的第一个元素，并且返回值是第一个元素的值
- RPOP key 弹出列表的最后一个元素，并且返回值是最后一个元素的值
- LLEN key 获取列表的元素个数
- LRANGE key start stop 获取下表[start, stop)的所有列表元素的值，start的最小值为0，并且支持负数下标
- LREM key count value, 当count > 0时，从列表左边开始，删除count个等于value的元素，当count < 0时，从列表右边开始，删除-count个等于value的元素，当count = 0时，删除所有等于value的元素

根据列表的特性，它非常使用于存储最近更新文章，因为最新更新是按照ID排序的,并且有时候文章删除后，会造成ID不连续,这样用这个获取最近10篇文章这样的功能就更合适了。

	import redis
	import json
	import time


	def create_a_post(client, author, title, date):
    	dict = {}
    	dict['author'] = author
    	dict['title'] = title
    	dict['date'] = date

    	id = client.incr('post-global-id', 1)
    	for k, v in dict.iteritems():
        	client.hset('post' + str(id), k, v)
    	client.lpush('last-posts', 'post' + str(id))
    	return id

	def get_a_post(client, id):
    
    	dict = client.hgetall('post' + str(id))
    	print dict
    	print id, 
    	for k,v in dict.iteritems():
    	    print v,

	def get_last_ten(client):
    	post_ids = client.lrange('last-posts',0,9)
    	print post_ids

	def main():
    	client = redis.StrictRedis()
    	id = create_a_post(client, 'Charles', 'a blog', time.time()) 
    	get_a_post(client, id)
    	get_last_ten(client)

	if __name__ == '__main__':
    	main()

在这个程序中，我们假设'post-global-id'已经初始化为0了。

###基础数据结构-集合

集合类型的基本命令如下：

- SADD key value1[value...] 添加多个值到key所在的集合
- SREM key value1[value...] 在key所在的集合删除多个列表
- SMEMBERS key 获取集合key中所有的元素
- SISMEMBER key value 判断value是否在key的集合中
- SDIFF key[key...] 集合N个集合的差集
- SINTER key[key...] 求N个集合的交集
- SUNION key[key...] 求N个集合的并集

集合非常适合博客中博文的标签，比如如果为一片文章添加多个标签可以

    SADD post-tag-42 C C++ git

这样就给ID为42的文章打上了tag标签，有时候我们也希望能通过标签查找所有对应的文章，这个也可以用集合实现，例如

    SADD tag-c 4 42 44

这样标签C语言对应的文章的ID列表有4 42 44

###基础数据结构-有序集合类型

有序集合类型的基本命令如下：

- ZADD key score value1 [score value...] 由于是有序集合类型，所以需要为每个值指定一个score，用来排序
- ZSCORE key value 获取key,value对应的score值
- ZRANGE key start stop [WITHSCORES] 获取start stop范围内的值列表
- ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count] 这个命令很好理解，可以获取key处于min和max之间的所有值
- ZINCRBY key score value 增加key中value的值score

如果需要对某个东西排序，并且需要经常访问其中的元素的时候使用，因为要求排序，则只能使用列表和有序集合，而如果要在大数据集合中访问某个元素，用列表是不适合的，因为列表不具有随机访问性，而有序列表的底层实现是用散列和跳跃表实现的，所以可以轻松的做到随机访问。
