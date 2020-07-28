### 1. 使用redis有哪些好处？ {#1-使用redis有哪些好处}

\(1\) 速度快，因为数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O\(1\)  
\(2\) 支持丰富数据类型，支持string，list，set，sorted set，hash  
\(3\) 支持事务，操作都是原子性，所谓的原子性就是对数据的更改要么全部执行，要么全部不执行  
\(4\) 丰富的特性：可用于缓存，消息，按key设置过期时间，过期后将会自动删除

### 2. redis相比memcached有哪些优势？ {#2-redis相比memcached有哪些优势}

\(1\) memcached所有的值均是简单的字符串，Redis作为其替代者，支持更为丰富的数据类型  
\(2\) redis的速度比memcached快很多  
\(3\) redis可以持久化其数据

### 3. Memcache与Redis的区别都有哪些？ {#3-memcache与redis的区别都有哪些}

1\)、存储方式  
Memecache把数据全部存在内存之中，断电后会挂掉，数据不能超过内存大小。  
Redis有部份存在硬盘上，这样能保证数据的持久性。  
2\)、数据支持类型  
Memcache对数据类型支持相对简单。  
Redis有复杂的数据类型。  
3\)、使用底层模型不同  
它们之间底层实现方式 以及与客户端之间通信的应用协议不一样。  
Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求。

4）value大小 redis最大可以达到1GB，而memcache只有1MB

### 4. redis常见性能问题和解决方案： {#4-redis常见性能问题和解决方案}

\(1\) Master最好不要做任何持久化工作，如RDB内存快照和AOF日志文件  
\(2\) 如果数据比较重要，某个Slave开启AOF备份数据，策略设置为每秒同步一次  
\(3\) 为了主从复制的速度和连接的稳定性，Master和Slave最好在同一个局域网内  
\(4\) 尽量避免在压力很大的主库上增加从库  
\(5\) 主从复制不要用图状结构，用单向链表结构更为稳定，即：Master &lt;- Slave1 &lt;- Slave2 &lt;- Slave3…  
这样的结构方便解决单点故障问题，实现Slave对Master的替换。如果Master挂了，可以立刻启用Slave1做Master，其他不变。

https://zhuanlan.zhihu.com/p/118561398?utm\_source=wechat\_session&utm\_medium=social&utm\_oi=988051857176055808

