### 字典
##### 简介
* 字典又称符号表，映射或关联数组，是一种保存键值对的抽象数据结构。
* Redis数据库的底层也是用字典实现的，对数据库的增删改查也是基于对字典的操作之上的。
* 字典还是哈希键的底层实现之一，当哈希键对比较多或者键值对中的元素都是比较长的，Redis就会使用字典作为底层实现。
##### 字典实现
**字典的实现是以哈希表作为它的底层实现，一个哈希表可以有多个哈希表节点，每个节点保存了字典中的一个键值对。**
1.哈希表节点
key就是键，v就是键中的值（可以是指针，unit64_t 整数， int64_t  s64整数），next是将另外一个哈希值相同的键值对连接在一起的指针（为了解决冲突）

```
typedef struct dictEntry{
   void *key;
   union{
   void *var;
   unit64_t u64;
   int64_t  s64;
   }v;
 struct dictEntry *next;
}
```
2.哈希表
table属性是一个数组，数组中每个元素都指向一个哈希表节点 ，每个哈希表节点都保存着一个键值对。
size记录了哈希表的大小，也就是table数组的大小。
used属性记录哈希表目前已有哈希表节点（键值对）的数量。
sizemask总是等于size-1（这个属性和哈希值一起决定一个键应该被放到table的那个索引上）。
```
typedef struct dictht{
   dictEntry **table；
   unsigned long size；
   unsigned long sizemask；
   unsigned long used；
}dictht；
```
3.字典
type和pribdata是配套的，针对不同类型的键值对，为创建多态字典而设置的。
type指向dictType结构的指针，每一个dictType里都保存了一簇用于操作特定类型键值对的函数（为用途不同的字典设置不同的类型特定函数）。
privdata保存了保存了需要传给那些类型特定函数的可选参数（也就是在dictType结构体中的参数）。
ht包含了两项数组，每个项都是dictht哈希表，字典只使用ht[0]哈希表，ht[1]只会在ht[0]rehash时使用，除了ht[1]之外另一个和rehash（重新散列）有关的就是rehashidx（记录rehash进度，若没在进行rehash则值为-1）.
```
typedef struct dict{
   dicType *type；
   void *pribdata；
   dictht  ht[2];
   int trehashidx;
}dict;
```
```
/* 保存一连串操作特定类型键值对的函数 */
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key); /* 哈希函数 */
    void *(*keyDup)(void *privdata, const void *key); /* 复制键函数 */
    void *(*valDup)(void *privdata, const void *obj); /* 复制值函数 */
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); /* 比较键函数 */
    void (*keyDestructor)(void *privdata, void *key); /* 销毁键函数 */
    void (*valDestructor)(void *privdata, void *obj); /* 销毁值函数 */
} dictType;
```
这就是字典实现的数据结构，如果要数据库性能好，还是要用一些性能较好的算法，Redis使用的MurmurHash2算法来计算键的哈希值（数据库的底层实现或者哈希键的底层实现）。
##### 哈希算法
在我们需要把一个新的键值对加入到字典里，程序得先根据键值对的键计算出哈希值和索引值，然后根据索引，将新键值对的哈希表节点方法哈希表数组的指定索引（位置）上，这都是由哈希算法来完成的。哈希算法的设计推理就不写了，因为这个是一个很复杂的过程。
Redis计算哈希值和索引值的方法

```
//用字典设置的哈希函数计算key的哈希值
hash = dict->type->hashFunction(key);
//利用哈希表的sizemask和哈希值来计算出索引值，h[x]可以是h[1]或h[0]因情况而定。
index = hash & dict->ht[x].sizemask;
```
这种算法的优点就是：对于输入有规律的键仍能给出一个很好的随机分布性而且计算速度也很快！但是呢，这种算法可能会出现冲突，因此要避免冲突就得由个解决冲突的办法？（在前面提到过）
##### 解决键冲突
什么是冲突：因为算法执行时会有可能多个键被分配到哈希数组的同一个索引上。
Redis中解决键冲突采用的是链地址法，每个节点都有一个next指针，构成冲突的节点可以用next指针构成一个单链表来共同占有同一个索引。这个解决方法也是解决冲突比较经典的方法，也是比较简单的方法。
##### rehash
rehash是什么？为什么要rehash？
rehash是重新散列的意思，因为在不断的执行中，哈希表保存的键值对在逐渐增多或减少，为了让哈希表的负载因子（load factor）维持在合理范围内，**当哈希表中的键值对过多或者过少时，需要对表的大小进行扩展或收缩**。
哈希表执行rehash的步骤：
1.为字典的ht[1]分配空间，此哈希表的大小取决于要执行的操作和h[0]包含的键值对的数量（ht[0].used）
 * 扩展操作：h[1]的大小等于第一个大于ht[0].used*2的2的n次方。
 * 收缩操作：h[1]的大小等于第一个大于ht[0].used的2的n次方。
 
2.将保存在ht[0]中的键值对到ht[1]上：重新计算哈希值和索引，然后将键值对放到ht[1]哈希表的指定位置上。
3. ht[0]迁移到ht[1]后，ht[0]变为空表然后释放掉，然后再将ht[1]设置为ht[0]，并再ht[1]位置上新建一个空白的哈希表，供下一次rehash使用。
##### 哈希表的自动扩展与收缩
如果在负载因子不合理时没有进行手动的rehash的话，那系统会在某些条件成立下自动进行扩展或收缩。
哈希表的负载因子求法：负载因子=以保存节点数量/哈希表大小
```
load_factor = ht[0].userd/ht[0].size
```
* 当系统满足以下的任意一个条件程序就会自动开始对哈希表执行扩展操作：
1.服务器目前没在执行BGSAVE命令或者BGREWRITEAOF命令并且哈希表的负载因子大于1
2.服务器目前正执行BGSAVE命令或者BGREWRITEAOF命令并且负载因子大于5
* 当系统满足负载因子小于0.1，就会自动进行收缩操作。
##### 渐进式rehash
其实在扩展或者收缩哈希表的时候并不是一次性，集中性的执行的，而是分多次，渐进式地完成的。
渐进式的详细步骤：
1.为ht[1]分配空间，此时字典同时有ht[0]和ht[1]两个哈希表。
2.在字典中维持一个索引计数器变量rehashidx，并设为0，表示rehash正式开始。
3.在rehash期间，每次对字典进行添加，删除，查找，更新时，程序除了执行指定操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]上，rehash完成，然后rehashidx+1。
4.直到全部内移到ht[1]，这时rehashidx属性的值设为-1，表示rehash操作完成。
采取分而治之的方式，将rehash键值对所需的工作均摊到对字典的增删改查上，避免了集中式rehash带来的庞大计算量。
##### 渐进式rehash执行期间的哈希表操作
因为在rehash期间字典会同时使用ht[0]和ht[1]，因此，增删改查会在两个哈希表上进行，比如查询操作，先对0表扫描如果没找到，就再从1表里找。注意的是，如果在此期间进行插入操作的话，那就会插入到1表，而不是0表。因为插入到0表就没意义了等于浪费体力。也保证了0表只减不增。




