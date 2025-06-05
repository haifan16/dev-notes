## Dict 的结构

Dict 类似 Java 的 HashTable，底层就是**数组+单向链表**的实现

Dict 包含两个哈希表，ht[0] 平常用，ht[1] 用来 rehash

## Dict 的扩容

当集合中元素较多时会出现哈希冲突增多的情况，导致链表过长，大大降低查询效率
-> Dict 在每次**新增**键值对时，会检查是否需要扩容

### 什么时候会触发全局哈希表扩容？

（负载因子：LoadFactor = used / size）

- **哈希表的加载因子 >= 1 并且没有执行 bgsave 或 bgrewriteaof 等后台进程**
- **哈希表的加载因子 > 5**

## Dict 的缩容

Dict 在每次**删除**键值对时，会检查是否需要缩容

### 什么时候会触发全局哈希收缩？

**当负载因子 < 0.1 时**

## Dict 的 rehash

不论是扩容还是收缩，都会创建新的哈希表，导致哈希表的 size 和 sizemask 变化。因此必须对哈希表中的每一个 key 重新计算索引，插入新的哈希表，这个过程称为 **rehash**。过程如下：

1. 计算新 hash 表的 realSize，值取决于当前要做的是扩容还是收缩：

- 如果是**扩容**，则新 size 为**第一个大于等于 dict.ht[0].used + 1 的 2^n**
- 如果是**缩容**，则新 size 为**第一个大于等于 dict.ht[0].used 的 2^n（不得小于 4）**

2. 按照新的 realSize 申请内存空间，创建 dictht，并赋值给 dict.ht[1]
3. 设置 dict.rehashidx = 0，标示开始 rehash
4. **将 dict.ht[0] 中的每一个 dictEntry 都 rehash 到 dict.ht[1]**
5. 将 dict.ht[1] 赋值给 dict.ht[0]，给 dict.ht[1] 初始化为空哈希表，释放原来的 dict.ht[0] 的内存

以上的过程存在什么问题呢？试想一下，如果 Dict 中包含数百万的 entry，要在一次 rehash 中完成，极有可能导致主线程阻塞。
-> 因此 **Dict 的 rehash 并不是一次性完成的，而是分多次、渐进式完成的（每次访问 Dict 时执行一次 rehash）**，称为**渐进式 rehash**。过程如下：

1. 计算新 hash 表的 realSize，值取决于当前要做的是扩容还是收缩：

- 如果是**扩容**，则新 size 为**第一个大于等于 dict.ht[0].used + 1 的 2^n**
- 如果是**缩容**，则新 size 为**第一个大于等于 dict.ht[0].used 的 2^n（不得小于 4）**

2. 按照新的 realSize 申请内存空间，创建 dictht，并赋值给 dict.ht[1]
3. 设置 dict.rehashidx = 0，标示开始 rehash
4. ~~将 dict.ht[0] 中的每一个 dictEntry 都 rehash 到 dict.ht[1]~~
    **每次执行新增、查询、修改、删除操作时，都检查一下 dict.rehashidx 是否大于 -1，如果是则将 dict.ht[0].table[rehashidx] 的 entry 链表 rehash 到 dict.ht[1]，并且 rehashidx++，直至 dict.ht[0] 的所有数据都 rehash 到 dict.ht[1]**
5. 将 dict.ht[1] 赋值给 dict.ht[0]，给 dict.ht[1] 初始化为空哈希表，释放原来的 dict.ht[0] 的内存
6. **将 rehashidx 赋值为 -1，代表 rehash 结束**
7. **在 rehash 过程中，新增操作直接写入 ht[1]，查询、修改、删除则会在 dict.ht[0] 和 dict.ht[1] 依次查找并执行 -> 确保 ht[0] 的数据只减不增，随着 rehash 的进行最终为空**