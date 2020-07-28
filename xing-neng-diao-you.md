## 1. 选择合适的存储引擎: InnoDB

## 2. 保证从内存中读取数据。将数据保存在内存中

#### 2.1 足够大的 innodb\_buffer\_pool\_size

推荐将数据全然保存在 innodb\_buffer\_pool\_size ，即按存储量规划 innodb\_buffer\_pool\_size 的容量。这样你能够全然从内存中读取数据。最大限度降低磁盘操作。

#### 2.2 数据预热

默认情况，仅仅有某条数据被读取一次，才会缓存在 innodb\_buffer\_pool。所以，数据库刚刚启动，须要进行数据预热，将磁盘上的全部数据缓存到内存中。

数据预热能够提高读取速度。

#### 2.3 不要让数据存到 SWAP 中

假设是专用 MYSQL server。能够禁用 SWAP，假设是共享server，确定 innodb\_buffer\_pool\_size 足够大。或者使用固定的内存空间做缓存，使用 memlock 指令。

## 3. 定期优化重建数据库

## 4. 降低磁盘写入操作

#### 4.1 使用足够大的写入缓存 innodb\_log\_file\_size

可是须要注意假设用 1G 的 innodb\_log\_file\_size 。假如server当机。须要 10 分钟来恢复。

推荐 innodb\_log\_file\_size 设置为 0.25 \* innodb\_buffer\_pool\_size

#### 4.2 innodb\_flush\_log\_at\_trx\_commit

这个选项和写磁盘操作密切相关：

innodb\_flush\_log\_at\_trx\_commit = 1 则每次改动写入磁盘  
innodb\_flush\_log\_at\_trx\_commit = 0/2 每秒写入磁盘

假设你的应用不涉及非常高的安全性 \(金融系统\)，或者基础架构足够安全，或者 事务都非常小，都能够用 0 或者 2 来减少磁盘操作。

#### 4.3 避免双写入缓冲

```
innodb_flush_method=O_DIRECT
```

## 5. 提高磁盘读写速度

RAID0 尤其是在使用 EC2 这样的虚拟磁盘 \(EBS\) 的时候，使用软 RAID0 很重要。

## 6. 充分使用索引



