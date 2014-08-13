###安装过程

首先，到redis的github release页面获取你需要的版本，例如[2.6.0](https://github.com/antirez/redis/releases/tag/2.6.0),具体的安装过程如下：

		wget https://github.com/antirez/redis/releases/tag/2.6.0
		tar -xvf 2.6.0.zip
        cd redis-2.6.0
        make 
        make install

如果安装顺利的话，在/usr/local/bin目录应该会有以下几个redis相关的二进制执行文件：

    [root@node]# ls redis-*
	redis-benchmark  redis-check-aof  redis-check-dump  redis-cli  redis-server

其中用的比较多的就属redis-server和redis-cli了，看名字也就知道他们分别是redis的服务器和客户端（自带命令行客户端）。

###配置和使用

**简单使用**

先来看看最简单的使用方法：redis-server内置了一些默认的配置，所有启动时不需要指定配置文件也可以运行，其命令如下：

    [root@node]# redis-server

命令执行之后，会起一个redis的服务端

然后，我们可以用命令行客户端redis-cli来连接它，由于上面使用的是默认的配置，那么，我们使用redis-cli的时候也不需要指定任何参数，具体命令如下：

    [root@node]# redis-cli    
	redis 127.0.0.1:6379> 

如果出现了"redis 127.0.0.1:6379>"的提示符就表示和服务端连接成功，可以开始执行命令了，这里我们看到服务端默认使用的端口号是6379。

我们输入一个ping命令，来看看服务器是否能正确应答：

    redis 127.0.0.1:6379> ping
	PONG
	redis 127.0.0.1:6379>

可以看出，服务器用PONG来响应了ping命令，说明客户端与服务端是正常连接的。

**配置**

redis-server启动的时候可以在命令行参数里面设置一个config文件来覆盖默认的配置，一个常见的配置文件如下：

[redis.conf](https://raw.githubusercontent.com/antirez/redis/2.6/redis.conf)

使用方式时

    [root@node]# redis-server /path/to/redis.conf

redis.conf里面涉及到的配置很多，但是他们都有一个基本的格式：

    关键字 参数1 参数2 ... 参数N

例如：

    slaveof 127.0.0.1:6380
其他的配置的也可以从上面的redis.conf文件的注释中获得

除了通过配置文件来配置，redis还支持通过命令行传参数的方式来配置，而且如果命令行参数和redis.conf文件不一致的话，会以命令行参数的配置为准。

**以服务的方式启动**

在linux下还可以通过服务的方式启动redis-server，在centos下面是写一个服务脚本放在/etc/init.d/下面，下面是我在github找到的一个centos下的redis服务脚本：

    #!/bin/sh
    #
    # redis - this script starts and stops the redis-server daemon
    #
    # chkconfig:   - 85 15 
    # description:  Redis is a persistent key-value database
    # processname: redis-server
    # config:  /etc/redis/redis.conf
    # config:  /etc/sysconfig/redis
    # pidfile: /var/run/redis.pid
     
    # Source function library.
    . /etc/rc.d/init.d/functions
     
    # Source networking configuration.
    . /etc/sysconfig/network
     
    # Check that networking is up.
    [ "$NETWORKING" = "no" ] && exit 0
     
    redis="/usr/local/bin/redis-server"
    prog=$(basename $redis)
     
    REDIS_CONF_FILE="/etc/redis/redis.conf"
     
    [ -f /etc/sysconfig/redis ] && . /etc/sysconfig/redis
     
    lockfile=/var/lock/subsys/redis
     
    start() {
    [ -x $redis ] || exit 5
    [ -f $REDIS_CONF_FILE ] || exit 6
    echo -n $"Starting $prog: "
    daemon $redis $REDIS_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
    }
     
    stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
    }
     
    restart() {
    stop
    start
    }
     
    reload() {
    echo -n $"Reloading $prog: "
    killproc $redis -HUP
    RETVAL=$?
    echo
    }
     
    force_reload() {
    restart
    }
     
    rh_status() {
    status $prog
    }
     
    rh_status_q() {
    rh_status >/dev/null 2>&1
    }
     
    case "$1" in
    start)
    rh_status_q && exit 0
    $1
    ;;
    stop)
    rh_status_q || exit 0
    $1
    ;;
    restart|configtest)
    $1
    ;;
    reload)
    rh_status_q || exit 7
    $1
    ;;
    force-reload)
    force_reload
    ;;
    status)
    rh_status
    ;;
    condrestart|try-restart)
    rh_status_q || exit 0
    	;;
    *)
    echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload}"
    exit 2
    esac
    
我们把配置好的redis.conf文件放到REDIS_CONF_FILE指定的目录下面，然后，就可以用service方式来启动，关闭redis了

    service redis start
    service redis stop








