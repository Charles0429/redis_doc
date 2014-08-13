###redis客户端

redis的客户端有很多，具体的请看[redis clients](http://redis.io/clients),我选了个python语言下推荐的客户端[redis.py](https://github.com/andymccurdy/redis-py)

在使用redis.py之前，先来讲讲怎么安装redis.py的

    wget https://github.com/andymccurdy/redis-py/archive/2.10.1.tar.gz
	tar -xvf 2.10.1.tar.gz
    cd redis-py-2.10.1
    python setup.py build 
    python setup.py install

测试安装是否成功

	[root@node redis-py-2.10.1]# python 
	Python 2.6.6 (r266:84292, Jan 22 2014, 09:42:36) 
	[GCC 4.4.7 20120313 (Red Hat 4.4.7-4)] on linux2
	Type "help", "copyright", "credits" or "license" for more information.
	>>> import redis
	>>> r = redis.StrictRedis(host='localhost', port=6379, db=0)
	>>> r.set('foo', 1)
	True
	>>> r.get('foo')
	'1'
	>>> 

如果上面的命令能够正确运行，那么说明redis.py安装成功了。

附redis.py的文档地址：http://redis-py.readthedocs.org/en/latest/

