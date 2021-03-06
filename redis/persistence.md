# Persistence

在开始介绍Redis的持久化之前，先对Redis的高可用相关的技术做一个概览：
- 持久化：是最简单的高可用方法，主要作用是数据的备份
- 主从复制：是Redis高可用的基础，哨兵和集群都是在主从复制的基础上实现的。主从复制实现了数据的多机备份，可以用于负载均衡以及简单的故障恢复。其缺点是故障恢复无法自动化；写操作无法负载均衡；存储能力受到单机的限制
- 哨兵：在复制的基础上，实现了自动化的故障恢复。缺点是写操作无法负载均衡；存储能力收到单机的限制
- 集群：解决了写操作无法负载均衡以及存储能力收到单机限制的问题，实现了较为完善的高可用方案



Redis提供两种方式的持久化：
- RDB：将Redis在内存中的数据记录定时dump到磁盘的文件上
- AOF：将Redis的写、删除数据的操作日志记录到磁盘文件上

## RDB

RDB持久化方式是通过快照完成的，Redis将内存中的所有数据以二进制的方式dump到磁盘的文件上，一般来说可以配置间隔多少时间就dump一次数据用于备份。

`save`和`bgsave`命令可以手动触发RDB持久化，区别在于：
- `save`命令使用Redis的进程进行RDB持久化，会直接阻塞Redis，这段时间不会响应任何用户发来的请求
- `bgsave`执行时，Redis会fork出一个子进程用于执行RDB持久化，fork的操作是阻塞的但比较短暂，后面RDB持久化的操作由子进程完成，不会阻塞处理用户请求的主进程

另外，在FLUSHALL、SHUTDOWN、主从复制的时候，Redis会自动触发RDB持久化。

当Redis意外崩溃或者关闭再次启动时，此时AOF持久化未开启时(默认未开启)，将使用RDB快照文件恢复数据。但是我们知道RDB一般是隔一段时间备份一次的，所以可能会丢掉一部分数据。

## AOF

AOF可以将Redis执行的每一条写命令追加到磁盘文件中，在Redis启动时优先选择从AOF文件中恢复数据。由于每一次的写操作，Redis都会记录到文件中，所以开启AOF持久化会对性能有一定的影响。

可以用`bgrewriteaof`手动触发AOF，也可以使用配置自动触发。

AOF文件重写过程与RDB快照bgsave工作过程有点相似，都是通过fork子进程，由子进程完成相应的操作，同样的在fork子进程简短的时间内，Redis是阻塞的。

## RDB vs AOF

RDB的优势是：
- 可以全部都备份到一个文件中
- 启动时，如果数据集很大，从RDB文件中恢复更快

RDB的缺点是：
- RDB都是定时持久化，Redis挂了之后会丢失一部分数据
- 数据集过大的时候，RDB持久化fork出来的子进程会抢占CPU资源，导致主进程收到比较大的影响

AOF的优势是：
- 与RDB持久化相比，AOF持久化数据丢失更少，其消耗内存更少(RDB方式执行bgsve会有内存拷贝)

AOF的缺点是：
- 对于相同数量的数据集而言，AOF文件通常要大于RDB文件
- AOF在运行效率上往往会慢于RDB


ps：Redis4.0版本可以开启混合持久化，同时启用RDB和AOF。