# Redis学习笔记

## Nosql发展和简介

### 数据库架构发展简介

先是从最原始的架构，单机版数据库存储一个应用的信息

![1558772436641](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558772436641.png)

但是上述架构下，有很多的存储瓶颈，比如数据库的总容量，数数据的索引，对应用的访问量等等

由于数据量和访问量的上升，架构逐渐会演变为下面对Mysql垂直拆分，并且加上Cache，减小Mysql压力。

![1558772585565](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558772585565.png)

但是这样的架构并不能有效的解决问题，为什么呢？缓存只能缓解读的压力，如果读和写都集中在一个数据库，那会让数据库处于危险状态，因此就有有了主从读写分离的架构演变。

![1558772766164](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558772766164.png)

把写的操作都放在主库，读的操作在从库中读取。但是这时候，主库的写压力不断增加，又因为使用了MyISAM使用表锁，高并发会出现严重锁的问题，然后就采用InnoDB使用它的行锁，来尝试替代MyISAM。

如果把数据都存在一个表上效率会特别慢，因此这时候大家都在想怎么可以让其他的库在承担一个表的压力，并且不会冲突（比如前1万在库1的表1上，后一万在库2的表1上实现分割）。因此在此同时分库分表成为了一个热门问题，集群主要是在可靠性上得到了保障。

![1558772992880](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558772992880.png)

然后就到了现在的这种架构，服务功能拆分，nginx负载均衡，再到多个Tomcat再到多个数据库等等。

![1558773390814](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558773390814.png)

### 什么是NoSql

NoSql=Not Only Sql（不仅仅是SQl），泛指为非关系性数据库，这些数据不需要固定的模式，比如创建关系型数据库表结构需要固定模式。比如一些高频信息，热点评论等都可能会用到Redis。

### NoSql优点

#### 易扩展

数据之间无联系，因此再扩展时不用考虑数据之间的关系。

#### 大数据量高性能

这些还是得益于它的数据间的无关系，数据库结构简单。

#### 数据模型多样灵活

再关系数据库中并且数据量大的情况下，对字段的增删是一个非常麻烦的事情。但是再NoSql不用提前建立字段，自定义数据格式。 

### 3V和三高

### 3V

* 海量（Volume）：数据量大
* 多样（Variety）：各种数据类型
* 实时（Velocity）：实时更新

### 3高

* 高并发：用法量大
* 高可扩：高可扩充（横向）
* 高性能：容灾，备份等等

## Nosql数据库的四大分类

### K V键值

比如Reids就是典型的K V键值对模式的还有Tokyo等等

### 文档型数据库

比如Mongdb，是一种介于关系型和非关系型的一种数据库，存储一些大的数据。是由c++编写。

### 列存储数据库

  比如Cassandra，HBase（和Hadoop有关），分布式文件系统。

### 图关系数据库

放的是关系，比如朋友圈社交网络，广告推广，如Neo4J等

## Nosql中的CAP

CAP的核心是CAP+BASE

### CAP理论

* c：Consistency（强一致性）：任何一个读操作总是能读取到之前完成的写操作结果，也就是在分布式环境中，多点的数据是一致的。
* a：Availablity（可用性）：好的可用性主要是指系统能够很好的为用户服务，不出现用户操作失败或者访问超时等用户体验不好的情况。可用性通常情况下可用性和分布式数据冗余，负载均衡等有着很大的关联。
* p：Partition tolerance（分取容错性）：分区容错性和扩展性紧密相关。在分布式应用中，可能因为一些分布式的原因导致系统无法正常运转。好的分区容错性要求能够使应用虽然是一个分布式系统，而看上去却好像是在一个可以运转正常的整体。比如现在的分布式系统中有某一个或者几个机器宕掉了，其他剩下的机器还能够正常运转满足系统需求，这样就具有好的分区容错性。

### BASE

* BA：Basically Available （基本可用）
* S：Soft state（软状态）
* E：Eventually consistent（最终一致性）

可以看出，CAP是保证性能优化，而BASE是保证基本可用，不管如何最终的结果一定要是正确的。

### CAP理论三进二

> CAP理论的核心就是一个分布式系统不可能同时满足CAP，不能认为Nosql数据库要同时满足CAP理论，最多只能同时较好的满足两个。

因此只能两两组合CA，CP，AP。

![1558793140818](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558793140818.png)

可以看到Redis和MongoDB，HBase都是满足CP，也就是强一致性和分区容忍（其实在分布式中分区容忍还是比较重要的）

![1558793184393](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558793184393.png)

![1558794002239](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558794002239.png)

现在的分布式一般都是牺牲了CP而保证了AP，因此CP就仅仅是一致性稍微弱类一点，毕竟可用才是最重要的。

### 分布式和集群的区别

![1558794565995](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558794565995.png)

## 正式进入Redis

### 什么是Redis

Redis：REmote DIctionary Server（远程字典服务器），是一个用c编写的高性能（key/value）分布式内存数据库。

Redis支持多种数据类型比如list，set，zset，hash，sds等等，并且也可以进行数据持久化，还有主从。

### Redis能干嘛

* 内存存储和持久化
* 模拟HttpSession这种需要摄动过期事件的功能
* 获取N个数据的操作，可以将数据ID放在Redis的List集合里面
* 可以使用Zset获得评论热度
* 发布，订阅消息系统
* 定时器，计数器

### Redis杂项

* 单进程，通过对epoll函数的包装，执行处理完全依靠主进程处理。
* 可以通过Select来切换库，一共16（0-15）个。
* Dbsize查看当前数据库的key的数量
* keys * 就相当于与select *，最好不要用keys *
* FLUSHDB清空当前库，FLUSHALL清空所有库
* 默认端口是6379
* Redis索引都是以0开始
* EXSISTS  key 可以判断某个key是否存在，返回1表示存在

### Redis关于key的操作

* keys * ：查看所有key
* move key db 从当前库中移除此key
* expire key （单位秒）：为某个key设置过期时间
* ttl key：查看还有多少秒过期
* type key：查看key是什么类型

![1558851109279](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558851109279.png)

![1558851621722](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558851621722.png)

## Redis五大数据类型

> 下边对五大数据类型常用的API进行演示！

### 字符串对象（String）

使用的是简单动态字符串（SDS，simple dynamic String）

* set/get/del/append/strlen

  ![1558852220321](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558852220321.png)

* Incr/decr/incrby/decrby（必须是整数才可以）

  ![1558852518826](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558852518826.png)

* getrange/setrange（获取/设定指定区间内的值）

  ![1558852990321](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558852990321.png)

* setex（set with expire）/setnx（set if not exist）

  ![1558854843488](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558854843488.png)

* mset（more set）/mget/msetnx：一次性加多个，set每次只能加/改一个

  ![1558855202708](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558855202708.png)

* getset（先get后set）

  ![1558855523434](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558855523434.png)

### 列表对象（List）

实际是一个LinkedList。

* lpush/rpush/lrange

  ![1558855787769](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558855787769.png)

* lpop/rpop

  ![1558855914630](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558855914630.png)

* lindex（按下标取）

  ![1558856074053](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558856074053.png)

* llen：查长度

  ![1558856192268](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558856192268.png)

* lrem key 删除N个value：可以删除集合中任意个数的Value

  ![1558856442973](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558856442973.png)

* ltrim key 开始index  结束index：截取指定范围了值

  ![1558857137788](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558857137788.png)

* rpoplpush（右出左进）

  ![1558857452894](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558857452894.png)

* lset key index value：修改指定索引的值

  ![1558857619595](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558857619595.png)

* linsert key before/after  value1 newValue

  ![1558857872072](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558857872072.png)

总结：list可以从左或从右插入，可如果键不存在就创建新的链表，已存在就新增内容，里面的内容全移除了，对应的键也就消失了，链表对头和尾操作效率比较高，中间较低。

### 集合对象（Set）

无序无重复，通过HashTable实现。

* sadd/smembers/sismember

  ![1558863215220](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558863215220.png)

* scard:获取集合中元素的个数

  ![1558863328870](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558863328870.png)

* srem key value 删除集合中的元素

  ![1558863419744](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558863419744.png)

* srandmember key  n（随机中出n个数）：从set中随机选n个数（比如可以用作挑选幸运观众）

  ![1558863606689](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558863606689.png)

* spop key ：随机出栈，因为set不是有序的

  ![1558864007285](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558864007285.png)

* smove key1 key2 在key1中的值：将key1的值移到key2里面

  ![1558864134339](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558864134339.png)

* 数学集合

  * sdiff key1 key2：差集（在key1里而不在key2里面）

    ![1558864177025](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558864177025.png)

  * sinter：交集

    ![1558864266396](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558864266396.png)

  * sunion：并集

    ![1558864317338](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558864317338.png)

### 有序集合对象（Zset，sorted set）

带分数的Set，在Set基础上加了Score。

* zadd/zrange【withscore】

  ![1558885311016](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558885311016.png)

  也可以加withscore同时打印分数

  ![1558885487096](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558885487096.png)

* zrangebyscore key 开始score  结束score 【limit index num】

  ![1558885894415](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558885894415.png)

  也可以使用limit来限定最多的显示的开始位置和数量（有些像分页）

  ![1558886353755](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558886353755.png)

* zrem key 某score下对应的value：删除元素

  ![1558886515785](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558886515785.png)

* zcard/zcount/zrank

  ![1558887249374](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558887249374.png)

* zrevrank key values：逆序获得下标值

  ![1558887444169](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558887444169.png)

* zrevrange：逆序排列

  ![1558887591169](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558887591169.png)

* zrevrangebyscore key 

  ![1558887750054](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558887750054.png)

### 哈希对象（Hash）

K V结构不变，但是其Value是一个键值对,相比于前面的，hash的Value是成对存在的。

* hset/hget/hmset/hmget/hgetall/hdel

  ![1558867413632](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558867413632.png)

* hlen key ：在key的hash中由多少个键值对

  ![1558867508119](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558867508119.png)

* hexists key key里面的某个值的key

  ![1558867587405](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558867587405.png)

* hkeys/hvals：查看某个key中包含所有键值对的key/value

  ![1558867694094](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558867694094.png)

* hincrby/hincrbyfloat：自增

  ![1558884764641](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558884764641.png)

* hsetnx（if not exist）

  ![1558884927496](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558884927496.png)

## Redis的配置文件

* 单位，和linux对大小写不敏感

  ![1558926499554](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558926499554.png)

* include：通过inclue配置不同的多个配置文件

  ![1558926605615](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558926605615.png)

* daemonize：设置是否进行后台运行

  ![1558927293061](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1558927293061.png)

* 指定pid所在的文件目录

  ```C
  # 当redis作为守护进程运行的时候，它会把 pid 默认写到 /var/run/redis.pid 文件里面，
  # 但是你可以在这里自己制定它的文件位置。
  pidfile /var/run/redis.pid
  ```

* 端口号：默认6379

  ```
  # 监听端口号，默认为 6379，如果你设为 0 ，redis 将不在 socket 上监听任何客户端连接。
  port 6379
   
  ```

* tcp-backlog

  ![1558927625371](C:\Users\Administrator\Documents\1558927625371.png)

  ```Redis
  # TCP 监听的最大容纳数量
  #
  # 在高并发的环境下，你需要把这个值调高以避免客户端连接缓慢的问题。
  # Linux 内核会一声不响的把这个值缩小成 /proc/sys/net/core/somaxconn 对应的值，
  # 所以你要修改这两个值才能达到你的预期。
  tcp-backlog 511
  ```

  等等

  下面是详细的配置文件

  ```Redis
  # redis 配置文件示例
   
  # 当你需要为某个配置项指定内存大小的时候，必须要带上单位，
  # 通常的格式就是 1k 5gb 4m 等酱紫：
  #
  # 1k  => 1000 bytes
  # 1kb => 1024 bytes
  # 1m  => 1000000 bytes
  # 1mb => 1024*1024 bytes
  # 1g  => 1000000000 bytes
  # 1gb => 1024*1024*1024 bytes
  #
  # 单位是不区分大小写的，你写 1K 5GB 4M 也行
   
  ################################## INCLUDES ###################################
   
  # 假如说你有一个可用于所有的 redis server 的标准配置模板，
  # 但针对某些 server 又需要一些个性化的设置，
  # 你可以使用 include 来包含一些其他的配置文件，这对你来说是非常有用的。
  #
  # 但是要注意哦，include 是不能被 config rewrite 命令改写的
  # 由于 redis 总是以最后的加工线作为一个配置指令值，所以你最好是把 include 放在这个文件的最前面，
  # 以避免在运行时覆盖配置的改变，相反，你就把它放在后面（外国人真啰嗦）。
  #
  # include /path/to/local.conf
  # include /path/to/other.conf
   
  ################################ 常用 #####################################
   
  # 默认情况下 redis 不是作为守护进程运行的，如果你想让它在后台运行，你就把它改成 yes。
  # 当redis作为守护进程运行的时候，它会写一个 pid 到 /var/run/redis.pid 文件里面。
  daemonize no
   
  # 当redis作为守护进程运行的时候，它会把 pid 默认写到 /var/run/redis.pid 文件里面，
  # 但是你可以在这里自己制定它的文件位置。
  pidfile /var/run/redis.pid
   
  # 监听端口号，默认为 6379，如果你设为 0 ，redis 将不在 socket 上监听任何客户端连接。
  port 6379
   
  # TCP 监听的最大容纳数量
  #
  # 在高并发的环境下，你需要把这个值调高以避免客户端连接缓慢的问题。
  # Linux 内核会一声不响的把这个值缩小成 /proc/sys/net/core/somaxconn 对应的值，
  # 所以你要修改这两个值才能达到你的预期。
  tcp-backlog 511
   
  # 默认情况下，redis 在 server 上所有有效的网络接口上监听客户端连接。
  # 你如果只想让它在一个网络接口上监听，那你就绑定一个IP或者多个IP。
  #
  # 示例，多个IP用空格隔开:
  #
  # bind 192.168.1.100 10.0.0.1
  # bind 127.0.0.1
   
  # 指定 unix socket 的路径。
  #
  # unixsocket /tmp/redis.sock
  # unixsocketperm 755
   
  # 指定在一个 client 空闲多少秒之后关闭连接（0 就是不管它）
  timeout 0
   
  # tcp 心跳包。
  #
  # 如果设置为非零，则在与客户端缺乏通讯的时候使用 SO_KEEPALIVE 发送 tcp acks 给客户端。
  # 这个之所有有用，主要由两个原因：
  #
  # 1) 防止死的 peers
  # 2) Take the connection alive from the point of view of network
  #    equipment in the middle.
  #
  # On Linux, the specified value (in seconds) is the period used to send ACKs.
  # Note that to close the connection the double of the time is needed.
  # On other kernels the period depends on the kernel configuration.
  #
  # A reasonable value for this option is 60 seconds.
  # 推荐一个合理的值就是60秒
  tcp-keepalive 0
   
  # 定义日志级别。
  # 可以是下面的这些值：
  # debug (适用于开发或测试阶段)
  # verbose (many rarely useful info, but not a mess like the debug level)
  # notice (适用于生产环境)
  # warning (仅仅一些重要的消息被记录)
  loglevel notice
   
  # 指定日志文件的位置
  logfile ""
   
  # 要想把日志记录到系统日志，就把它改成 yes，
  # 也可以可选择性的更新其他的syslog 参数以达到你的要求
  # syslog-enabled no
   
  # 设置 syslog 的 identity。
  # syslog-ident redis
   
  # 设置 syslog 的 facility，必须是 USER 或者是 LOCAL0-LOCAL7 之间的值。
  # syslog-facility local0
   
  # 设置数据库的数目。
  # 默认数据库是 DB 0，你可以在每个连接上使用 select <dbid> 命令选择一个不同的数据库，
  # 但是 dbid 必须是一个介于 0 到 databasees - 1 之间的值
  databases 16
   
  ################################ 快照 ################################
  #
  # 存 DB 到磁盘：
  #
  #   格式：save <间隔时间（秒）> <写入次数>
  #
  #   根据给定的时间间隔和写入次数将数据保存到磁盘
  #
  #   下面的例子的意思是：
  #   900 秒内如果至少有 1 个 key 的值变化，则保存
  #   300 秒内如果至少有 10 个 key 的值变化，则保存
  #   60 秒内如果至少有 10000 个 key 的值变化，则保存
  #　　
  #   注意：你可以注释掉所有的 save 行来停用保存功能。
  #   也可以直接一个空字符串来实现停用：
  #   save ""
   
  save 900 1
  save 300 10
  save 60 10000
   
  # 默认情况下，如果 redis 最后一次的后台保存失败，redis 将停止接受写操作，
  # 这样以一种强硬的方式让用户知道数据不能正确的持久化到磁盘，
  # 否则就会没人注意到灾难的发生。
  #
  # 如果后台保存进程重新启动工作了，redis 也将自动的允许写操作。
  #
  # 然而你要是安装了靠谱的监控，你可能不希望 redis 这样做，那你就改成 no 好了。
  stop-writes-on-bgsave-error yes
   
  # 是否在 dump .rdb 数据库的时候使用 LZF 压缩字符串
  # 默认都设为 yes
  # 如果你希望保存子进程节省点 cpu ，你就设置它为 no ，
  # 不过这个数据集可能就会比较大
  rdbcompression yes
   
  # 是否校验rdb文件
  rdbchecksum yes
   
  # 设置 dump 的文件位置
  dbfilename dump.rdb
   
  # 工作目录
  # 例如上面的 dbfilename 只指定了文件名，
  # 但是它会写入到这个目录下。这个配置项一定是个目录，而不能是文件名。
  dir ./
   
  ################################# 主从复制 #################################
   
  # 主从复制。使用 slaveof 来让一个 redis 实例成为另一个reids 实例的副本。
  # 注意这个只需要在 slave 上配置。
  #
  # slaveof <masterip> <masterport>
   
  # 如果 master 需要密码认证，就在这里设置
  # masterauth <master-password>
   
  # 当一个 slave 与 master 失去联系，或者复制正在进行的时候，
  # slave 可能会有两种表现：
  #
  # 1) 如果为 yes ，slave 仍然会应答客户端请求，但返回的数据可能是过时，
  #    或者数据可能是空的在第一次同步的时候
  #
  # 2) 如果为 no ，在你执行除了 info he salveof 之外的其他命令时，
  #    slave 都将返回一个 "SYNC with master in progress" 的错误，
  #
  slave-serve-stale-data yes
   
  # 你可以配置一个 slave 实体是否接受写入操作。
  # 通过写入操作来存储一些短暂的数据对于一个 slave 实例来说可能是有用的，
  # 因为相对从 master 重新同步数而言，据数据写入到 slave 会更容易被删除。
  # 但是如果客户端因为一个错误的配置写入，也可能会导致一些问题。
  #
  # 从 redis 2.6 版起，默认 slaves 都是只读的。
  #
  # Note: read only slaves are not designed to be exposed to untrusted clients
  # on the internet. It's just a protection layer against misuse of the instance.
  # Still a read only slave exports by default all the administrative commands
  # such as CONFIG, DEBUG, and so forth. To a limited extent you can improve
  # security of read only slaves using 'rename-command' to shadow all the
  # administrative / dangerous commands.
  # 注意：只读的 slaves 没有被设计成在 internet 上暴露给不受信任的客户端。
  # 它仅仅是一个针对误用实例的一个保护层。
  slave-read-only yes
   
  # Slaves 在一个预定义的时间间隔内发送 ping 命令到 server 。
  # 你可以改变这个时间间隔。默认为 10 秒。
  #
  # repl-ping-slave-period 10
   
  # The following option sets the replication timeout for:
  # 设置主从复制过期时间
  #
  # 1) Bulk transfer I/O during SYNC, from the point of view of slave.
  # 2) Master timeout from the point of view of slaves (data, pings).
  # 3) Slave timeout from the point of view of masters (REPLCONF ACK pings).
  #
  # It is important to make sure that this value is greater than the value
  # specified for repl-ping-slave-period otherwise a timeout will be detected
  # every time there is low traffic between the master and the slave.
  # 这个值一定要比 repl-ping-slave-period 大
  #
  # repl-timeout 60
   
  # Disable TCP_NODELAY on the slave socket after SYNC?
  #
  # If you select "yes" Redis will use a smaller number of TCP packets and
  # less bandwidth to send data to slaves. But this can add a delay for
  # the data to appear on the slave side, up to 40 milliseconds with
  # Linux kernels using a default configuration.
  #
  # If you select "no" the delay for data to appear on the slave side will
  # be reduced but more bandwidth will be used for replication.
  #
  # By default we optimize for low latency, but in very high traffic conditions
  # or when the master and slaves are many hops away, turning this to "yes" may
  # be a good idea.
  repl-disable-tcp-nodelay no
   
  # 设置主从复制容量大小。这个 backlog 是一个用来在 slaves 被断开连接时
  # 存放 slave 数据的 buffer，所以当一个 slave 想要重新连接，通常不希望全部重新同步，
  # 只是部分同步就够了，仅仅传递 slave 在断开连接时丢失的这部分数据。
  #
  # The biggest the replication backlog, the longer the time the slave can be
  # disconnected and later be able to perform a partial resynchronization.
  # 这个值越大，salve 可以断开连接的时间就越长。
  #
  # The backlog is only allocated once there is at least a slave connected.
  #
  # repl-backlog-size 1mb
   
  # After a master has no longer connected slaves for some time, the backlog
  # will be freed. The following option configures the amount of seconds that
  # need to elapse, starting from the time the last slave disconnected, for
  # the backlog buffer to be freed.
  # 在某些时候，master 不再连接 slaves，backlog 将被释放。
  #
  # A value of 0 means to never release the backlog.
  # 如果设置为 0 ，意味着绝不释放 backlog 。
  #
  # repl-backlog-ttl 3600
   
  # 当 master 不能正常工作的时候，Redis Sentinel 会从 slaves 中选出一个新的 master，
  # 这个值越小，就越会被优先选中，但是如果是 0 ， 那是意味着这个 slave 不可能被选中。
  #
  # 默认优先级为 100。
  slave-priority 100
   
  # It is possible for a master to stop accepting writes if there are less than
  # N slaves connected, having a lag less or equal than M seconds.
  #
  # The N slaves need to be in "online" state.
  #
  # The lag in seconds, that must be <= the specified value, is calculated from
  # the last ping received from the slave, that is usually sent every second.
  #
  # This option does not GUARANTEES that N replicas will accept the write, but
  # will limit the window of exposure for lost writes in case not enough slaves
  # are available, to the specified number of seconds.
  #
  # For example to require at least 3 slaves with a lag <= 10 seconds use:
  #
  # min-slaves-to-write 3
  # min-slaves-max-lag 10
  #
  # Setting one or the other to 0 disables the feature.
  #
  # By default min-slaves-to-write is set to 0 (feature disabled) and
  # min-slaves-max-lag is set to 10.
   
  ################################## 安全 ###################################
   
  # Require clients to issue AUTH <PASSWORD> before processing any other
  # commands.  This might be useful in environments in which you do not trust
  # others with access to the host running redis-server.
  #
  # This should stay commented out for backward compatibility and because most
  # people do not need auth (e.g. they run their own servers).
  # 
  # Warning: since Redis is pretty fast an outside user can try up to
  # 150k passwords per second against a good box. This means that you should
  # use a very strong password otherwise it will be very easy to break.
  # 
  # 设置认证密码
  # requirepass foobared
   
  # Command renaming.
  #
  # It is possible to change the name of dangerous commands in a shared
  # environment. For instance the CONFIG command may be renamed into something
  # hard to guess so that it will still be available for internal-use tools
  # but not available for general clients.
  #
  # Example:
  #
  # rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
  #
  # It is also possible to completely kill a command by renaming it into
  # an empty string:
  #
  # rename-command CONFIG ""
  #
  # Please note that changing the name of commands that are logged into the
  # AOF file or transmitted to slaves may cause problems.
   
  ################################### 限制 ####################################
   
  # Set the max number of connected clients at the same time. By default
  # this limit is set to 10000 clients, however if the Redis server is not
  # able to configure the process file limit to allow for the specified limit
  # the max number of allowed clients is set to the current file limit
  # minus 32 (as Redis reserves a few file descriptors for internal uses).
  #
  # 一旦达到最大限制，redis 将关闭所有的新连接
  # 并发送一个‘max number of clients reached’的错误。
  #
  # maxclients 10000
   
  # 如果你设置了这个值，当缓存的数据容量达到这个值， redis 将根据你选择的
  # eviction 策略来移除一些 keys。
  #
  # 如果 redis 不能根据策略移除 keys ，或者是策略被设置为 ‘noeviction’，
  # redis 将开始响应错误给命令，如 set，lpush 等等，
  # 并继续响应只读的命令，如 get
  #
  # This option is usually useful when using Redis as an LRU cache, or to set
  # a hard memory limit for an instance (using the 'noeviction' policy).
  #
  # WARNING: If you have slaves attached to an instance with maxmemory on,
  # the size of the output buffers needed to feed the slaves are subtracted
  # from the used memory count, so that network problems / resyncs will
  # not trigger a loop where keys are evicted, and in turn the output
  # buffer of slaves is full with DELs of keys evicted triggering the deletion
  # of more keys, and so forth until the database is completely emptied.
  #
  # In short... if you have slaves attached it is suggested that you set a lower
  # limit for maxmemory so that there is some free RAM on the system for slave
  # output buffers (but this is not needed if the policy is 'noeviction').
  #
  # 最大使用内存
  # maxmemory <bytes>
   
  # 最大内存策略，你有 5 个选择。过期策略
  # 
  # volatile-lru -> remove the key with an expire set using an LRU algorithm
  # volatile-lru -> 使用 LRU 算法移除包含过期设置的 key 。
  # allkeys-lru -> remove any key accordingly to the LRU algorithm
  # allkeys-lru -> 根据 LRU 算法移除所有的 key 。
  # volatile-random -> remove a random key with an expire set
  # allkeys-random -> remove a random key, any key
  # volatile-ttl -> remove the key with the nearest expire time (minor TTL)
  # noeviction -> don't expire at all, just return an error on write operations
  # noeviction -> 不让任何 key 过期，只是给写入操作返回一个错误
  # 
  # Note: with any of the above policies, Redis will return an error on write
  #       operations, when there are not suitable keys for eviction.
  #
  #       At the date of writing this commands are: set setnx setex append
  #       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
  #       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
  #       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
  #       getset mset msetnx exec sort
  #
  # The default is:
  #
  # maxmemory-policy noeviction
   
  # LRU and minimal TTL algorithms are not precise algorithms but approximated
  # algorithms (in order to save memory), so you can tune it for speed or
  # accuracy. For default Redis will check five keys and pick the one that was
  # used less recently, you can change the sample size using the following
  # configuration directive.
  #
  # The default of 5 produces good enough results. 10 Approximates very closely
  # true LRU but costs a bit more CPU. 3 is very fast but not very accurate.
  #
  # maxmemory-samples 5
   
  ############################## APPEND ONLY MODE ###############################
   
  # By default Redis asynchronously dumps the dataset on disk. This mode is
  # good enough in many applications, but an issue with the Redis process or
  # a power outage may result into a few minutes of writes lost (depending on
  # the configured save points).
  #
  # The Append Only File is an alternative persistence mode that provides
  # much better durability. For instance using the default data fsync policy
  # (see later in the config file) Redis can lose just one second of writes in a
  # dramatic event like a server power outage, or a single write if something
  # wrong with the Redis process itself happens, but the operating system is
  # still running correctly.
  #
  # AOF and RDB persistence can be enabled at the same time without problems.
  # If the AOF is enabled on startup Redis will load the AOF, that is the file
  # with the better durability guarantees.
  #
  # Please check http://redis.io/topics/persistence for more information.
   
  appendonly no
   
  # The name of the append only file (default: "appendonly.aof")
   
  appendfilename "appendonly.aof"
   
  # The fsync() call tells the Operating System to actually write data on disk
  # instead to wait for more data in the output buffer. Some OS will really flush 
  # data on disk, some other OS will just try to do it ASAP.
  #
  # Redis supports three different modes:
  #
  # no: don't fsync, just let the OS flush the data when it wants. Faster.
  # always: fsync after every write to the append only log . Slow, Safest.
  # everysec: fsync only one time every second. Compromise.
  #
  # The default is "everysec", as that's usually the right compromise between
  # speed and data safety. It's up to you to understand if you can relax this to
  # "no" that will let the operating system flush the output buffer when
  # it wants, for better performances (but if you can live with the idea of
  # some data loss consider the default persistence mode that's snapshotting),
  # or on the contrary, use "always" that's very slow but a bit safer than
  # everysec.
  #
  # More details please check the following article:
  # http://antirez.com/post/redis-persistence-demystified.html
  #
  # If unsure, use "everysec".
   
  # appendfsync always
  appendfsync everysec
  # appendfsync no
   
  # When the AOF fsync policy is set to always or everysec, and a background
  # saving process (a background save or AOF log background rewriting) is
  # performing a lot of I/O against the disk, in some Linux configurations
  # Redis may block too long on the fsync() call. Note that there is no fix for
  # this currently, as even performing fsync in a different thread will block
  # our synchronous write(2) call.
  #
  # In order to mitigate this problem it's possible to use the following option
  # that will prevent fsync() from being called in the main process while a
  # BGSAVE or BGREWRITEAOF is in progress.
  #
  # This means that while another child is saving, the durability of Redis is
  # the same as "appendfsync none". In practical terms, this means that it is
  # possible to lose up to 30 seconds of log in the worst scenario (with the
  # default Linux settings).
  # 
  # If you have latency problems turn this to "yes". Otherwise leave it as
  # "no" that is the safest pick from the point of view of durability.
   
  no-appendfsync-on-rewrite no
   
  # Automatic rewrite of the append only file.
  # Redis is able to automatically rewrite the log file implicitly calling
  # BGREWRITEAOF when the AOF log size grows by the specified percentage.
  # 
  # This is how it works: Redis remembers the size of the AOF file after the
  # latest rewrite (if no rewrite has happened since the restart, the size of
  # the AOF at startup is used).
  #
  # This base size is compared to the current size. If the current size is
  # bigger than the specified percentage, the rewrite is triggered. Also
  # you need to specify a minimal size for the AOF file to be rewritten, this
  # is useful to avoid rewriting the AOF file even if the percentage increase
  # is reached but it is still pretty small.
  #
  # Specify a percentage of zero in order to disable the automatic AOF
  # rewrite feature.
   
  auto-aof-rewrite-percentage 100
  auto-aof-rewrite-min-size 64mb
   
  ################################ LUA SCRIPTING  ###############################
   
  # Max execution time of a Lua script in milliseconds.
  #
  # If the maximum execution time is reached Redis will log that a script is
  # still in execution after the maximum allowed time and will start to
  # reply to queries with an error.
  #
  # When a long running script exceed the maximum execution time only the
  # SCRIPT KILL and SHUTDOWN NOSAVE commands are available. The first can be
  # used to stop a script that did not yet called write commands. The second
  # is the only way to shut down the server in the case a write commands was
  # already issue by the script but the user don't want to wait for the natural
  # termination of the script.
  #
  # Set it to 0 or a negative value for unlimited execution without warnings.
  lua-time-limit 5000
   
  ################################ REDIS 集群  ###############################
  #
  # 启用或停用集群
  # cluster-enabled yes
   
  # Every cluster node has a cluster configuration file. This file is not
  # intended to be edited by hand. It is created and updated by Redis nodes.
  # Every Redis Cluster node requires a different cluster configuration file.
  # Make sure that instances running in the same system does not have
  # overlapping cluster configuration file names.
  #
  # cluster-config-file nodes-6379.conf
   
  # Cluster node timeout is the amount of milliseconds a node must be unreachable 
  # for it to be considered in failure state.
  # Most other internal time limits are multiple of the node timeout.
  #
  # cluster-node-timeout 15000
   
  # A slave of a failing master will avoid to start a failover if its data
  # looks too old.
  #
  # There is no simple way for a slave to actually have a exact measure of
  # its "data age", so the following two checks are performed:
  #
  # 1) If there are multiple slaves able to failover, they exchange messages
  #    in order to try to give an advantage to the slave with the best
  #    replication offset (more data from the master processed).
  #    Slaves will try to get their rank by offset, and apply to the start
  #    of the failover a delay proportional to their rank.
  #
  # 2) Every single slave computes the time of the last interaction with
  #    its master. This can be the last ping or command received (if the master
  #    is still in the "connected" state), or the time that elapsed since the
  #    disconnection with the master (if the replication link is currently down).
  #    If the last interaction is too old, the slave will not try to failover
  #    at all.
  #
  # The point "2" can be tuned by user. Specifically a slave will not perform
  # the failover if, since the last interaction with the master, the time
  # elapsed is greater than:
  #
  #   (node-timeout * slave-validity-factor) + repl-ping-slave-period
  #
  # So for example if node-timeout is 30 seconds, and the slave-validity-factor
  # is 10, and assuming a default repl-ping-slave-period of 10 seconds, the
  # slave will not try to failover if it was not able to talk with the master
  # for longer than 310 seconds.
  #
  # A large slave-validity-factor may allow slaves with too old data to failover
  # a master, while a too small value may prevent the cluster from being able to
  # elect a slave at all.
  #
  # For maximum availability, it is possible to set the slave-validity-factor
  # to a value of 0, which means, that slaves will always try to failover the
  # master regardless of the last time they interacted with the master.
  # (However they'll always try to apply a delay proportional to their
  # offset rank).
  #
  # Zero is the only value able to guarantee that when all the partitions heal
  # the cluster will always be able to continue.
  #
  # cluster-slave-validity-factor 10
   
  # Cluster slaves are able to migrate to orphaned masters, that are masters
  # that are left without working slaves. This improves the cluster ability
  # to resist to failures as otherwise an orphaned master can't be failed over
  # in case of failure if it has no working slaves.
  #
  # Slaves migrate to orphaned masters only if there are still at least a
  # given number of other working slaves for their old master. This number
  # is the "migration barrier". A migration barrier of 1 means that a slave
  # will migrate only if there is at least 1 other working slave for its master
  # and so forth. It usually reflects the number of slaves you want for every
  # master in your cluster.
  #
  # Default is 1 (slaves migrate only if their masters remain with at least
  # one slave). To disable migration just set it to a very large value.
  # A value of 0 can be set but is useful only for debugging and dangerous
  # in production.
  #
  # cluster-migration-barrier 1
   
  # In order to setup your cluster make sure to read the documentation
  # available at http://redis.io web site.
   
  ################################## SLOW LOG ###################################
   
  # The Redis Slow Log is a system to log queries that exceeded a specified
  # execution time. The execution time does not include the I/O operations
  # like talking with the client, sending the reply and so forth,
  # but just the time needed to actually execute the command (this is the only
  # stage of command execution where the thread is blocked and can not serve
  # other requests in the meantime).
  # 
  # You can configure the slow log with two parameters: one tells Redis
  # what is the execution time, in microseconds, to exceed in order for the
  # command to get logged, and the other parameter is the length of the
  # slow log. When a new command is logged the oldest one is removed from the
  # queue of logged commands.
   
  # The following time is expressed in microseconds, so 1000000 is equivalent
  # to one second. Note that a negative number disables the slow log, while
  # a value of zero forces the logging of every command.
  slowlog-log-slower-than 10000
   
  # There is no limit to this length. Just be aware that it will consume memory.
  # You can reclaim memory used by the slow log with SLOWLOG RESET.
  slowlog-max-len 128
   
  ############################# Event notification ##############################
   
  # Redis can notify Pub/Sub clients about events happening in the key space.
  # This feature is documented at http://redis.io/topics/keyspace-events
  # 
  # For instance if keyspace events notification is enabled, and a client
  # performs a DEL operation on key "foo" stored in the Database 0, two
  # messages will be published via Pub/Sub:
  #
  # PUBLISH __keyspace@0__:foo del
  # PUBLISH __keyevent@0__:del foo
  #
  # It is possible to select the events that Redis will notify among a set
  # of classes. Every class is identified by a single character:
  #
  #  K     Keyspace events, published with __keyspace@<db>__ prefix.
  #  E     Keyevent events, published with __keyevent@<db>__ prefix.
  #  g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
  #  $     String commands
  #  l     List commands
  #  s     Set commands
  #  h     Hash commands
  #  z     Sorted set commands
  #  x     Expired events (events generated every time a key expires)
  #  e     Evicted events (events generated when a key is evicted for maxmemory)
  #  A     Alias for g$lshzxe, so that the "AKE" string means all the events.
  #
  #  The "notify-keyspace-events" takes as argument a string that is composed
  #  by zero or multiple characters. The empty string means that notifications
  #  are disabled at all.
  #
  #  Example: to enable list and generic events, from the point of view of the
  #           event name, use:
  #
  #  notify-keyspace-events Elg
  #
  #  Example 2: to get the stream of the expired keys subscribing to channel
  #             name __keyevent@0__:expired use:
  #
  #  notify-keyspace-events Ex
  #
  #  By default all notifications are disabled because most users don't need
  #  this feature and the feature has some overhead. Note that if you don't
  #  specify at least one of K or E, no events will be delivered.
  notify-keyspace-events ""
   
  ############################### ADVANCED CONFIG ###############################
   
  # Hashes are encoded using a memory efficient data structure when they have a
  # small number of entries, and the biggest entry does not exceed a given
  # threshold. These thresholds can be configured using the following directives.
  hash-max-ziplist-entries 512
  hash-max-ziplist-value 64
   
  # Similarly to hashes, small lists are also encoded in a special way in order
  # to save a lot of space. The special representation is only used when
  # you are under the following limits:
  list-max-ziplist-entries 512
  list-max-ziplist-value 64
   
  # Sets have a special encoding in just one case: when a set is composed
  # of just strings that happens to be integers in radix 10 in the range
  # of 64 bit signed integers.
  # The following configuration setting sets the limit in the size of the
  # set in order to use this special memory saving encoding.
  set-max-intset-entries 512
   
  # Similarly to hashes and lists, sorted sets are also specially encoded in
  # order to save a lot of space. This encoding is only used when the length and
  # elements of a sorted set are below the following limits:
  zset-max-ziplist-entries 128
  zset-max-ziplist-value 64
   
  # HyperLogLog sparse representation bytes limit. The limit includes the
  # 16 bytes header. When an HyperLogLog using the sparse representation crosses
  # this limit, it is converted into the dense representation.
  #
  # A value greater than 16000 is totally useless, since at that point the
  # dense representation is more memory efficient.
  # 
  # The suggested value is ~ 3000 in order to have the benefits of
  # the space efficient encoding without slowing down too much PFADD,
  # which is O(N) with the sparse encoding. The value can be raised to
  # ~ 10000 when CPU is not a concern, but space is, and the data set is
  # composed of many HyperLogLogs with cardinality in the 0 - 15000 range.
  hll-sparse-max-bytes 3000
   
  # Active rehashing uses 1 millisecond every 100 milliseconds of CPU time in
  # order to help rehashing the main Redis hash table (the one mapping top-level
  # keys to values). The hash table implementation Redis uses (see dict.c)
  # performs a lazy rehashing: the more operation you run into a hash table
  # that is rehashing, the more rehashing "steps" are performed, so if the
  # server is idle the rehashing is never complete and some more memory is used
  # by the hash table.
  # 
  # The default is to use this millisecond 10 times every second in order to
  # active rehashing the main dictionaries, freeing memory when possible.
  #
  # If unsure:
  # use "activerehashing no" if you have hard latency requirements and it is
  # not a good thing in your environment that Redis can reply form time to time
  # to queries with 2 milliseconds delay.
  #
  # use "activerehashing yes" if you don't have such hard requirements but
  # want to free memory asap when possible.
  activerehashing yes
   
  # The client output buffer limits can be used to force disconnection of clients
  # that are not reading data from the server fast enough for some reason (a
  # common reason is that a Pub/Sub client can't consume messages as fast as the
  # publisher can produce them).
  #
  # The limit can be set differently for the three different classes of clients:
  #
  # normal -> normal clients
  # slave  -> slave clients and MONITOR clients
  # pubsub -> clients subscribed to at least one pubsub channel or pattern
  #
  # The syntax of every client-output-buffer-limit directive is the following:
  #
  # client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
  #
  # A client is immediately disconnected once the hard limit is reached, or if
  # the soft limit is reached and remains reached for the specified number of
  # seconds (continuously).
  # So for instance if the hard limit is 32 megabytes and the soft limit is
  # 16 megabytes / 10 seconds, the client will get disconnected immediately
  # if the size of the output buffers reach 32 megabytes, but will also get
  # disconnected if the client reaches 16 megabytes and continuously overcomes
  # the limit for 10 seconds.
  #
  # By default normal clients are not limited because they don't receive data
  # without asking (in a push way), but just after a request, so only
  # asynchronous clients may create a scenario where data is requested faster
  # than it can read.
  #
  # Instead there is a default limit for pubsub and slave clients, since
  # subscribers and slaves receive data in a push fashion.
  #
  # Both the hard or the soft limit can be disabled by setting them to zero.
  client-output-buffer-limit normal 0 0 0
  client-output-buffer-limit slave 256mb 64mb 60
  client-output-buffer-limit pubsub 32mb 8mb 60
   
  # Redis calls an internal function to perform many background tasks, like
  # closing connections of clients in timeout, purging expired keys that are
  # never requested, and so forth.
  #
  # Not all tasks are performed with the same frequency, but Redis checks for
  # tasks to perform accordingly to the specified "hz" value.
  #
  # By default "hz" is set to 10. Raising the value will use more CPU when
  # Redis is idle, but at the same time will make Redis more responsive when
  # there are many keys expiring at the same time, and timeouts may be
  # handled with more precision.
  #
  # The range is between 1 and 500, however a value over 100 is usually not
  # a good idea. Most users should use the default of 10 and raise this up to
  # 100 only in environments where very low latency is required.
  hz 10
   
  # When a child rewrites the AOF file, if the following option is enabled
  # the file will be fsync-ed every 32 MB of data generated. This is useful
  # in order to commit the file to the disk more incrementally and avoid
  # big latency spikes.
  aof-rewrite-incremental-fsync yes
  ```

  一般在那个目录下运行redis，日志等信息就有可能写在这个目录下。
  
  ![1559118861853](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559118861853.png)

![1559118885958](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559118885958.png)

Redis的删除策略配置文件，默认是noeviction（永不过期），LRU是最近最少，TTL是根据时间

![1559119260389](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559119260389.png)

### Redis常用配置

stdout：标准输出设备

![1559119401728](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559119401728.png)

![1559119528130](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559119528130.png)

![1559119555051](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559119555051.png)

## Redis的持久化

### RDB（Redis DataBase）

指定时间间隔内将内存中的数据集快照写入磁盘，行话就叫Snapshot快照，恢复时是将快照文件直接读到内存。

Redis会单独创建（fork，也就是复制一个和父进程一样的子进程）一个子进程来进行持久化，会先写到一个临时文件，等待持久化完成再进行替换。如果是大规模数据恢复，且数据完整性不是很敏感，那么RDB比AOF高效。RDB缺点就是最后一次持久化后的数据可能会丢失。

从配置文件中可以看到默认是是

![1559120569861](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559120569861.png)

* 900秒改过1次（15分钟）
* 300秒改过10次（5分钟）
* 60秒改过10000次（1分钟）

只要满足其中一个可以触发RDB持久化。当然可以从文档中看到设置为save "",把RDB禁用掉。也可以在命令行需要立刻将一个数据备份，就下一条输入save就行。

当然这些都是可以在配置文件配置的。默认的RDB文件在启动文件夹下名为Dump.rdb

![1559121840425](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559121840425.png)

当后台保存出错停止写

![1559121655785](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559121655785.png)

可以使用config get dir获得目录

![1559123091223](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559123091223.png)

#### RDB的缺点

1. 在一定时间间隔做一次备份，如果Rdis意外dowm，就会造成最后一次的快照后的所有修改数据
2. Fork的时候，内存数据被克隆了一份，大致两倍的膨胀性。

![1559122063421](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559122063421.png)

### AOF（Append Only File）

 是根据文件追加来实现的，以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来（不记录读指令），那么下一次Redis启动前会读取该文件重新构建数据到上一次写的地方。

![1559123014278](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559123014278.png)

在这块有个坑，在写操作的时候最后一句要是flushall的话，它也会追加，因此下次依旧看不到数据。
![1559123329724](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559123329724.png)

如果AOF文件损坏就不能启动起来，因此也可以看出AOF是先于RDB加载的，如果出现这种情况可以使用redis-check-aof来自动修复（将不符合规范的全部丢弃掉）

![1559123719657](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559123719657.png)

#### AOF数据同步

* Always：同步持久，每次变化就被立即记录，但是性能较差，但完整性较高
* Everysec：每间隔多长时间，默认一秒检查一次
* No

#### AOF恢复

1. 正常恢复：将appendnonly no改为yes，或者从别处复制一份数据，然后重启
2. 异常恢复：redis-check-aof

#### Rewrite

Rewrite是AOF采用文件追加方式，文件会越来越大Redis为避免这种情况，新增了rewrite机制，当AOF文件大小超过设定的阈值，Redis就会启动AOF文件的内容压缩，只保留可恢复数据的最小指令集可以使用bgrewriteaof。

重写的原理：

会fork一条新的进程来讲文件重写（先写到临时文件，最后再Rename），**注意并不是读取以前的AOF文件，而是根据当前数据库状态来生成最简的命令来达成一样的效果。**

触发机制：

**会记录上次重写的AOF大小，默认配置是的当AOF文件大小是上次rewrite后大小的一倍，且文件大小超过64M**

![1559124967726](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559124967726.png)

#### AOF优势

可以每修改同步同步（appendfsync always），每秒同步，不同步。

#### 缺点

* 相同的文件要存储空间要大于RDB文件，恢复慢于RDB
* AOF运行效率慢于RDB

![1559125282791](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559125282791.png)

#### 建议

![1559125433293](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559125433293.png)

![1559125483168](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559125483168.png)

可以发现后边会有主从复制解决AOF的一些问题。

## Redis事务

可以一次执行多个命令，是一组命令的集合，按顺序执行。Redis也支持CAS。

### 常用命令

* DISCARD：取消事务，放弃执行事务内的所有命令
* EXEC：执行所有事务块内的命令
* MULTI：标记一个事务的开始
* UNWATCHED：取watch命令对所有key的监视
* WATCH key 【...】:如果事务执行之前这些key被改动，那么事务讲被打断。

使用Multi开始事务，EXEC执行，DISCARD放弃执行。

正常执行：

![1559127118355](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559127118355.png)

放弃执行：

![1559127170194](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559127170194.png)

如果过程中因为语法错误也不能执行，整个事务就会执行失败。getset再事务执行中就立即报错。

![1559127448513](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559127448513.png)

如果语法正确，但是对值的操作不正确，结果只是不正确的记录会执行失败，其余会执行成功，我给字符串自增，只导致错的一天不会执行。

![1559127570282](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559127570282.png)

因此，Redis是对事务的部分支持，并不是强支持。

![1559127769186](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559127769186.png)悲观锁：认为会出事，每次做最坏的打算，相当于表锁

乐观锁：认为不会出事，每次做最好的打算，相当于行锁，并发度高，Redis中采用CAS来实现乐观锁。

Watch演示：

比如我们要去消费，money代表余额，spend代表花出去的钱，可以模拟一下场景

此时是一个初始状态，余额100，花费0，先尝试花费执行成功

![1559133583538](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559133583538.png)

此时呢，对money进行监视，我们开两个终端窗口，一个正在执行事务，一个正修改money。可以发现是执行失败的，其实还是采用的CAS。

![1559134818658](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559134818658.png)

**一旦使用了unwatch，那么之前加的所有watch都会被取消。**

## Redis的发布和订阅

虽然Redis有这种功能，但是基本不会有很多人用，因为都采取一些消息中间件。

使用subscribe订阅频道，使用publish进行消息发布

![1559136022084](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559136022084.png)

当然也可以是使用通配符对一类消息进行统配

![1559136153825](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559136153825.png)

这块就不多说，以后会学习RabbitMq或RocketMq。

## Redis主从复制（Master/Slave）

主从复制就是，主机数据更新后根据配置策略，自动同步到备机的master/slaver机制，Master以写为主，Slaver以读为主。

主要是用于读写分离和容灾恢复。

### Redis主从用法

1. 配从（库）不配主（库）

2. 从库配置：slaveof  主库IP  主库端口

   每次和master断开后都需要重新连接，除非配置进redis.conf文件中.

### 主从中的重要知识

一仆二主：就是一个主机，多个从机，主机要是挂了，就暂时没主机，可以使用后边的反客为主来替换，等待主机恢复，就又和从机自动连接上了，但是如果从机挂了，下一次需要重新声明主从关系。



![1559226114808](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559226114808.png)

![1559226627254](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559226627254.png)

**主从之间的复制原理**

![1559227115040](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559227115040.png)

**哨兵模式**

其实就是反客为主的自动版，会有一个哨兵巡逻，如果主机挂了，就选举一个从机作为主机

![1559227430999](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559227430999.png)

启动哨兵

![1559227527477](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1559227527477.png)

如果原来的主机被哨兵模式选举出来的主机取代了，那就是永久的。