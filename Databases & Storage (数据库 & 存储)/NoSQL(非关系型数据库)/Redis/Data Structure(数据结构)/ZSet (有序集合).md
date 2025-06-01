## ZSet 基础

ZSet = SortedSet，每个元素都需要指定一个 score 值和 member 值
- 自动根据 score 值排序
- member 必须唯一
- 可以根据 member 查询分数

```
ZADD z1 10 m1 20 m2 30 m3
```

根据 member 查询 score：
```
ZSCORE z1 m1
```

（TODO：补充）

## ZSet 的底层实现

ZSet 会根据数据量的大小使用不同的数据结构进行存储：
在**数据量较小**时使用**压缩列表**；
在**数据量较大**时采用**跳表 + 字段**结构。

1. 小数据量场景，使用压缩列表（ziplist）

当满足以下两个条件时，ZSet 使用压缩列表作为底层实现：
- 元素数量 < zset-max-ziplist-entries（默认值：128）
- 每个元素的大小 < zset-max-ziplist-value（默认值：64 字节）

压缩列表结构的特点：

是一种类似数组的结构，使用一段连续的内存空间存储数据。每个节点之间相邻存放，通过内存偏移量实现节点之间的关系（而不是通过指针），从而节省内存空间。

（TODO：补充）