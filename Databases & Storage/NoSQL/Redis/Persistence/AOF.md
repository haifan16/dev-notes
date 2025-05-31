## 概念

AOF(Append Only File)，追加文件

Redis 处理的每一个写命令都会记录在 AOF 文件，可以看作是**命令日志文件**。

## 配置

### 开启

AOF 默认是关闭的，需要在 redis.conf 中配置开启：

```
# 是否开启 AOF 功能，默认 no
appendonly yes
# AOF 文件的名称
appendfilename "appendonly.aof"
```

### 配置 AOF 的命令记录频率

```
# 每执行一次写命令，立即记录到 AOF 文件（几乎不丢数据，性能影响大）
appendfsync always
# 写命令执行完先放入 AOF 缓冲区，然后每隔 1 秒讲缓冲区数据写到 AOF 文件（默认方案）
appendfsync everysec
# 写命令执行完先放入 AOF 缓冲区，由操作系统决定什么时候将缓冲区内容写回磁盘（性能最好，可能丢失大量数据）
appendfsync no
```

## bgrewriteaof 命令

因为是记录命令，AOF 文件会比 RDB 文件大得多；而且 AOF 会记录对同一个 key 的多次写操作，但只有最后一次写操作才有意义。
-> 可以通过执行 bgrewriteaof 命令，让 AOF 文件执行重写功能，用最少的命令达到相同效果。

Redis 也会在触发阈值时自动重写 AOF 文件。阈值配置：

```
# AOF 文件比上次文件增长超过多少百分比则触发重写
auto-aof-rewrite-percentage 100
# AOF 文件体积最小多大以上才触发重写
auto-aof-rewrite-min-size 64mb
```