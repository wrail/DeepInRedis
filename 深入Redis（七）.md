# Redis的持久化

## RDB持久化

### 什么是RDB持久化？

* RDB持久化就是将Redis在内存中产生的数据保存在磁盘里，防止数据的丢失。（Redis是一个内存数据库，一旦进程退出，数据库状态也就消失不见了）

* RDB文件是一个二进制文件，保存的是某一个时间点的数据状态。

----------

### RDB的文件结构

完整的RDB文件包含以下部分

|REDIS|db_version|databases|EOF|check_sum|

* REDIS：长度五个字节，保存的REDIS这几个字母（不是字符串），RDB文件的标识符。

* db_version:四个字节记录RDB的版本号。

* database：保存0个或多个数据库，和各个数据库的键值对数据。

* EOF：一字节，标志RDB文件正文结束。

* check_sum:八字节无符号整数，由前四部份计算得来，用于检查RDB文件是否由出错或者受损的状况。

database中存不同的类型值方式是不同的，下边是数据库的结构

|SELECTDB|db_number|key_value-pairs|

* SELECTDB:是一个长度为1字节的常量，它表示下一个要读的是数据库号码

* db_number:保存的是数据库号码

* key_value_pairs:保存当前数据库的所有键值对，过期的也会保存在里面，根据过期种种原因，因此长度也不同。
  * 没过期的键值对由|TYPE|key|value|三部分组成。
  
  * key是一个字符串对象
  
  * value的结构和长度和TYPE有关。
   
  * 带有过期键的在RDB由|EXPIRETIME_MS|TYPE||key|value|,EXPIRETIME标识的是接下来要读的是一个过期时间以毫秒为单位。  

上边说到value的值是不定的，下面对每一种情况进行分析

1. 字符串对象：字符串对象的编码可以是int也可以是row，最大是32位的int（由8位，16位，32位）。如果是row的话，如果字符串的长度小于等于20，原样保存。如果大于20，就会被压缩（LZF算法）后再保存。

2. list对象：按照|list_length|item1|item2|...|itemN|保存。

3. 集合对象：按照|set_size|elem1|elem2|...|elemN|保存。

4. 哈希表对象：按照|hash_size|key1|value1|key2|value2|...|keyN|valueN|

5. 有序集合：按照|sorted_set_size|member1|score1|member2|score2|...|memberN|scoreN|

6. 整数集合（INTSET）：先将整数集合化为字符串对象保存再RDB文件中，在读入RDB文件时，根据TYPE值，读入此字符串，然后将此字符串转为原来的整数集合对象。

7.ZIPLIST，Zip_hashtable，Zset_Ziplist(有序集合)：使用ZipList编码的，value就是一个压缩列表。这种文件存入RDB的步骤是，先将压缩列表换为一个字符串对象，并存入RDB。读入的时候将它转换为原来的压缩列表对象（因为全都是压缩列表，不存在其他类型），然后根据TYPE的值来设置压缩列表对象的类型。

----------


### RDB文件的生成
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331104331754.png)

* SAVE方法：会阻塞Redis服务器的进程，直到RDB文件创建完毕。在此阻塞期间，服务器不能处理任何命令请求。

* BGSAVE方法：会派生出一个子进程，子进程负责创建RDB文件（完成或出错时会给父进程发信号），父进程继续处理命令。然后父进程一直轮询，等待子进程的信号。 在BGSAVE执行期间，客户端发送的SAVE，和BGSAVE会被服务器拒绝以免发生竞争（SAVE命令父进程和子进程同时调用rdbSAVE，BGSAVE子进程和子进程之间同时调用rdbSAVE）。

* 设置条件并自动间隔性保存：Redis允许通过设置save选项来让服务器每隔一段事件自动执行BGSAVE命令（在满足设置的条件的情况下）。条件保存在reidserver中的*saveparams结构体数组里。

> 在redis除了saveparams数组还有几个重要的属性，dirty计数器，记录距离上次成功执行SAVE或者BGSAVE命令后服务器对所有数据库进行了多少次修改。每一条修改命令都会让它加一。

> lastsave是一个UNIX时间戳，记录上一次SAVE或BGSAVE执行成功的事件。

> serverCron是一个周期性操作函数，每个100毫秒执行一次，用于维护数据库，使命之一就是检查save的条件是否满足，如果满足就执行BGSAVE。

----------


### RDB文件的载入和AOF文件的载入比较

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331104247646.png)

* RDB文件在服务器启动时自动载入，只要检测到RDB文件就自动载入并处于阻塞状态，直到载入完成。

* AOF文件的优先级高于RDB文件，如果服务器开启了AOF持久化功能，则在重启时会加载AOF文件，而不是RDB。（AOF通常更新频率高于RDB文件）

* 如果AOF持久化关闭，才回去找RDB文件。  

----------

### 进一步分析RDB文件

清空并保存为RDB文件。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331104440494.png)

打开已存的RDB文件，虽然是乱码也隐隐约约能看到REDIS字符串，version，ctime，还有一些常量。（空的RDB文件由REDIS字符串，version，EOF常量和Check_Sum组成）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331104606342.png)



存一个值后会明显发现下面多了一个字段（mesg）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331105353847.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331105255239.png)

虽然是乱码还是能大致的发现和我们前边说的结构大致相同。（有值的数据库有三部分组成，SELECTDB，db_number,key_value_pairs）

包含一个带有过期时间的字符串键的RDB文件。（带有过期键的RDB文件由EXPIRETIME_MS,ms,TYPE,key,value组成)

