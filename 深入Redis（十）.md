# Redis客户端

> 在前面介绍I/O多路复用的时候也说过客户端和服务器的对应关系，一个服务器根据套接字和多个客户端进行网络通信（单线程单进程）。

## Redis客户端

Redis客户端的状态信息都被存在了redisServer（结构体）的clients（链表）里了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402155326130.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

每一个client节点都有当前客户端的状态信息。

> 状态信息就比如说客户端套接字描述，客户端名字，客户端要执行的命令信息，复制状态，执行发布和订阅的数据结构等和客户端有关的东西


## 深入了解客户端

### 客户端里的属性

1. 通用属性：也就相当于提供一些足以让客户端可以完成一些基本操作的属性。

2. 特定属性： 比如要整形一些特殊功能会用到的属性，比如操纵数据库等。 

### 客户端的状态信息分析

由于保证这些的都是一些redisClient的一些特定属性，可能在后边会省略掉redisClient。

*  套接字描述符
 
 在redisClient中定义了一个fd（int类型），这个值可以为-1或者是一个大于-1的整数。

 伪客户端：fd的值为-1，伪客户端在前面也有说到过（持久化），它的请求主要是来源于AOF或者是Lua脚本（不联网），在AOF中用于载入并还原数据库状态，在Lua中用来执行Lua脚本中包含的Redis命令。

 普通客户端：fa的值大于-1，使用套接字和服务器交互（合法的套接字描述符不能为-1）。

* 客户端名字

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402162245117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

     > 1.默认情况下的普通客户端是没有名字的
 
     > 2.可以使用CLIEBT setname来设置名字  


* 客户端的标志值

在redisClient中flags属性代表客户端的标志值，标志值可以设置单个也可以设置多个。
  
    > flags = <flag>
    > flags = <flag1>|<flag2>|...

这些标志一部分标志记录客户端的角色，另一部分是记录客户端目前所处的状态。下边列出几个，如果需要看详细的话可以在官网或者自己搜一搜。
     
    REDIS_MASTER:标志客户端代表着一个主服务器
    REDIS_SLAVE:标志客户端代表的是一个从服务器
    REDIS_MONITOR:标志客户端正在执行monitor命令
    ..............

比如下面这句命令的意思就是:是专门用于执行LUA脚本的伪客户端，强制将当前执行的命令写入AOF文件，并强制复制给从服务器。

REDIS_LUA_CLIENT | REDIS_FORCE_AOF | REDIS_FORCE_REPL（为了使主服务器和从服务器东可以正常的载入SCRIPT LOAD命令指定的脚本）


* 输入缓冲区

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402171958732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

在redisClient中用querybuf（SDS）表示，主要用于保存客户端发送的命令请求。

客户端执行的命令会被计如querybuf，比如set k v 命令，它就会用SDS来记住这个命令和添加的元素。这个缓冲区的大小是可以调整的，最大**不能超过1G**，否则服务器就会强制关闭这个客户端。
 
* 命令与命令参数

argv是用来记录命令数组，对应的argc是用来记录argv的长度。如下图：

下图中的set k v 是一条命令

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402172648846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)
 
* 命令的实现函数

在上边的命令中取出命令并实现，用cmd属性来表示，如下图：

从上一步avg中取到命令，并在字典里查找命令并放入到redisCommand执行（图中以rpush为例）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402173558114.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

* 输出缓冲区

前面有一个输入缓冲区，当然也就对应有输出缓冲区，输出缓冲区保存的是执行命令所得的答复。有两个输出缓冲区可以用。一个是固定的，一个是可变的。

固定的缓冲区是用来保存那些长度比较小的恢复，而可变的缓冲区主要是用来保存那些长度较大的回复比如很大的集合，字符串等。

* 身份验证

在redisClient的authenticated属性用来记录客户端是否通过了身份验证。

如果authenticated的值为0，那就是代表客户端没有通过身份验证，如果authenticated的值为1，就代表通过了身份验证。当然，这仅仅是开启身份验证的时候才能起作用，如果没开身份验证，默认值为0，服务器也不会拒接客户端的请求。

* 时间

ctime：记录了创建客户端的时间。

lastinteraction：记录了客户端和服务端最后一次进行互动（interaction）的时间。

obuf_soft_limit_reached_time:记录了缓冲区第一次到达软性限制的时间。

### 客户端的创建

如果客户端是通过网络连接的，那就和前面的一样，把新的redisClient加到服务端属性clients链表的后边。如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402232203364.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

如果是一个伪客户端的话就不连接，单独创建。

### 关闭普通客户端

可能导致普通客户的关闭的几个因素：

1. 客户端进程被杀死，或者客户端进程退出。
2. 带有不符合协议格式的命令请求。
3. 客户端成为CLIENT KILL对象也会关闭。
4. 如果客户端空转超出了timeout设置的时间也会关闭。
5. 客户端发送命令的请求大小超过了输入缓存区的最大容量1G。
6. 服务器回复给客户端的答复大小超过最大的输出缓存区的大小也会关闭

在Redis中，理论上来讲一个输出缓存区的大小可以是很大的，但是为了比秒客户端由于回复过大而占用太多的资源，Redis中对输出缓存设置了两种限制模式。

1. 硬性模式模式：如果输出超过的Reids设置的硬性要求，客户端就会立马关闭。

2. 软性限制模式：如果超出了软性大小，但是又没超过硬性大小，服务器就会是能够用一个在前面说到过的一个obuf_soft_limit_reached_time来记录客户端到达软性限制的起始时间。如果在指定的时间一直超过软性限制所设置的大小（所谓的超负荷运转），那就会关闭客户端，否则就将obuf_soft_limit_reached_time置为0.

> lua会在服务器初始化时创建一个负责执行LUA脚本中包含的Redis命令的伪客户端，这个伪客户端在服务器的生命周期一直存在，伴随着服务器的关闭而关闭。

> AOF的伪客户端会在重启时写完AOF文件后关闭。

