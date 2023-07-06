# clickhouse mergetree结构解析
## merge tree文件构成
### 创建表
```
CREATE TABLE default.test
(
    `a` Int32,
    `b` Int32,
    `c` Int32,
    INDEX `idx_c` (c) TYPE minmax GRANULARITY 1
)
ENGINE = MergeTree
PARTITION BY a 
ORDER BY b
SETTINGS index_granularity=3，min_bytes_for_wide_part=0
```
### 插入数据
```
insert into default.test(a,b,c) values(1,1,1);
insert into default.test(a,b,c) values(5,2,2),(5,3,3);
insert into default.test(a,b,c) values(3,10,4),(3,9,5),(3,8,6),(3,7,7),(3,6,8),(3,5,9),(3,4,10);
```
### 磁盘文件
![](../images/image.png)

生成了 3 个数据目录，每个目录在称作一个分区(part)，目录名的前缀正是我们写入时字段 a 的值: 1,3,5，按照创建表时候`PARTITION BY a ` 。
查看a=3的分区如下：

![](../images/image-1.png)

checksums.txt：校验文件，使用二进制格式文件存储。保存了余下各类文件（primary.idx、count.txt等）的大小以及size的哈希值，用来快速校验文件的完整性。

columns.txt：列文件信息，明文存储

count.txt：计数文件，明文存储，记录当前当前分区目录下总行数

.bin 是列数据文件，使用压缩格式存储默认采用LZ4

*.mrk2 mark 文件，目的是快速定位 bin 文件数据位置

minmax_a.idx 分区键 min-max 索引文件，目的是加速分区键 a 查找

primay.idx 主键索引文件，目的是加速主键 b 查找

skp_idx_idx_c.* 字段 c 索引文件，目的是加速 c 的查找

### 数据和索引
#### granule
mergetree把bin文件根据颗粒度划分为多个颗粒（granule），把每个颗粒单独压缩存储。
`SETTINGS index_granularity=3`表示每 ３ 行数据为一个 granule，分区目前只有 ７ 条数据，所以被划分成 3 个 granule(三个色块)
![](../images/image-2.png)

为方便读取某个 granule，使用 *.mrk 文件记录每个 granule 的 offset，每个 granule 的 header 里会记录一些元信息，用于读取解析:

![](../images/image-3.png)

这样，我们就可以根据 ｍark 文件，直接定位到想要的 granule，然后对这个单独的 granule 进行读取、校验。

#### 稀疏索引
主键索引，可通过[PRIMARY KEY expr]指定，默认是 ORDER BY 字段值

#### skipping index
`INDEXidx_c(c) TYPE minmax GRANULARITY 1` 针对字段 c 创建一个 minmax 模式索引。
GRANULARITY 是稀疏点选择上的 granule 颗粒度，GRANULARITY 1 表示每 1 个 granule 选取一个

![](../images/image-4.png)

#### partition minmax index

针对分区键，MergeTree 还会创建一个 min/max 索引，来加速分区选择

![](../images/image-5.png)

### 分区和合并规则
目前产生了两个分区目录5_2_2_0、5_3_3_0。执行一下语句强制分区合并`optimize table test`

![](../images/image-6.png)

通过上图可知，我们强制触发表合并后生成了一个新的分区数据目录 5_2_3_1

![](../images/image-7.png)

MergeTree的分区目录是在数据写入过程中被创建的。也就是一张新建的表，如果没有任何数据，那么也不会有任何的分区存在。

MergeTree的分区目录伴随着每一批数据的写入(一次insert语句)，mergetree都会生成一批新的分区目录，即便不同批次写入的数据属于相同分区，也会生成不同的分区目录

每次insert都会产生一个分区，那么就存在多个相同分区的情况，clickhouse会通过后台任务再将相同分区的多个目录合并成一个新的目录。已经存在的旧目录并不会立即被删除，而是在之后的某个时刻通过后台被删除

同属于一个分区的多个目录，在合并之后会形成一个全新的目录，目录中的索引和数据文件也会相应的进行合并。新目录名称的合并规则如下

MinBlockNum: 取同一分区内所有目录中最小的MinBlockNum值

MaxBlockNum: 取同一分区内所有目录中最大的MaxBlockNum值

Level: 取同一分区内最大Level值并加1