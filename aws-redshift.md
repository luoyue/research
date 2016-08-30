## redshift

亚马逊的服务，支持PB级别数据的数据仓库

### 基本结构
#### cluster
* 一个cluster里面的所有机器都必须是同一个类型，可以增加同个类型机器的数量，也可以把所有的机器升级到同一个配置
* compute node对外界是一个黑箱子
* cluster resize的时候是read only的

#### leader node
* 当cluster node大于一台后，会自动分配一个leader node。
* 负责处理执行计划，编译代码，并把执行计划和编译后的代码分发到计算节点，而且只分发到表所在的计算节点。
* 所有的SQL Functions，都是在leader node上执行的，一些查询了数据表，又执行sql function的可能会报错，比如select current_schema(), userid from users;
* 计算节点查询完后，把数据汇总到leader node

#### compute node
* 每个计算节点又被分成多个slice，leader node分发执行计划的时候是分给每个slice的，查询也是以slice为单位，并行进行的。选择distribute key，可以让该列的数据在slice中均匀分布，从而提高并行查询的效率
* 基本大部分的操作都以slice为单位，有单独的cpu，内存，硬盘
* 计算优化型，1个cpu，1个slice；存储优化型，2个cpu，1个slice

#### 特点
* 分slice，提高查询速度
* 列式存储，可以在单个data block里面存储更多行的数据，但是要避免select *，尽量只查需要的字段
* 数据压缩，减少硬盘存储，减少硬盘io
* 查询优化器，在leader node进行
* 代码在leader node编译，在compute node不用再次编译
* 带宽很高，leader node与compute node之间high-speed network

### 索引和字段
#### dist key, distribution style 决定数据在slice如何分布，分三种style
* even distribution：默认方式，round-robin的策略，第一条存到slice1，第二条存到slice2....，join查询的时候可能会带来大量不同node之间的数据传输，但总体所有slicce的工作负载比较均匀
* all distribution：表数据在每个compute node都保存一份，避免不同node之间的数据传输，但明显会造成存储的压力
* key distribution：根据值选择slice，比如1开头的存slice1，2开头的存slice2，这样对join查询，把join的字段设置成key distribution，就可以保证要join的数据都在同一个slice内，避免数据传输

#### 使用
每个表只可以设置一个dist key，建好表后无法修改dist key，当主表有多个字段需要join的时候，就应该考量哪个join的表数据量最大，那么设置主表和这个表的这个字段为dist key，key distribution，其他的表join字段设置为dist key，all distribution
* 只有用来join的字段才需要设为dist key，key distribution
* 如果经常join某一列，则选择这一列做dist key
* 如果某一列经常做等值过滤，就不要选择做dist key，可以保证用到所有的slice进行并行计算查找
* 如果表数据很大，并且不会有join操作，就不要选择做分区索引

#### sort key
* single sort key：sortkey (id)，单个字段索引
* compound sort key：compound sortkey (id, type, chan)，多字段组合索引，使用必须按顺序id，type，chan，对type chan的组合无法过滤，类似mysql
* interleaved sort key：interleaved sortkey(id,type,chan)，多字段组合索引，<b>对其中字段的任意组合都可以命中索引</b>，有点厉害

#### 使用
* 最近的数据查询量很大，设timestamp字段提升为sort key 
* 对一个字段经常使用条件或者范围查询，设这个字段为sort key
* 经常join table，设这个字段为两个表的sort key

#### 字段的支持
支持unique key和primary key，但这两个都不保证数据的唯一性，插入相同的数据并不会报错
不能增加和删除 sort key，dist key
**无法修改字段类型**，只能通过加字段，copy旧字段数据到新字段，删除旧字段，rename新字段的方式来操作

###其他

####数据压缩
支持多种压缩，raw，bytedict，lzo，runlength，text255，text32k，对不同数据类型压缩效率不一样，可以使用analyze compression tablename，获得每个字段推荐的压缩格式，建好后不能更改

####集群resize
修改集群大小的时候，相当于建了一个新集群，把旧集群的数据复制到新的集群，整个过程集群会处于只读的状态。数据传输的速度比较慢，实测在50-100M/s，1T数据的集群可能要好几个小时才能完成。亚马逊文档有推荐加快resize的方法

http://docs.aws.amazon.com/zh_cn/redshift/latest/mgmt/rs-tutorial-using-snapshot-restore-resize-operations.html#rs-tutorial-copy-data-to-target-cluster

整个过程是

1. 建原cluster的快照
2. 用快照建新cluster
3. resize 新cluster
4. 把开始建快照到完成新cluster resize这段过程中，更新到原cluster的数据也更新到新cluster。这步推荐的方法是把这段时间的更新数据都上传到s3，然后在新的cluster copy s3的数据
5. 修改配置到新cluster
整个操作比较适合一段较长的时间load大量数据的情景，不太适合小段时间实时有更新的情景

#### 查询最佳实践
* 建表要建好
* 避免使用select *
* 尽量避免多次查询
* 尽量多写where条件，尽量带上primary key，sort key
* 尽量用直接比较的查询条件，= > < in,is true，少使用函数
* join操作，table2 join table1，where table1.key=table2.key and table1.id>'100' and table2.id>'100'，最后的条件看似多余但redshift还是有用
* group by尽量用sort key，像mysql组合索引一样保证顺序，group by sort1,sort2,sort3，而不是1和3
* sort by和group by顺序一致sort by a, b, c group by a, b, c
* redshift是鼓励使用子查询的，尽量避免多次查询

#### WLM工作负载管理
可以设置Queue里面查询的并发数，用户组，查询分组，内存使用上限，并发数可以在查询的时候动态调整

#### 权限控制
和mysql类似，创建用户，创建分组，设置user或者分组对某个schema的权限，可以建超级用户，加到createuser分组就行

####优化
修改，删除，增加大量数据，执行vacuum;analyze;前者可以恢复删除数据所用的硬盘，重新存储sort order，后者更新静态的元数据，利于查询优化器优化查询计划
redshift是鼓励使用子查询的，尽量避免多次查询

#### 常用sql：
* 查询所有database信息：select * from pg_database;
* 查询所有schema信息：select * from pg_namespace;
* 查询所有table信息：select * from pg_tables;
* 查询所有table结构信息：select * from pg_table_def;
* 查询所有用户信息：select * from pg_user;
* 查询所有分组：select * from pg_group;

#### 查询单表用的存储大小，单位M
```
select stv_tbl_perm.name as table, count(*) as mb
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name in ('lineorder','part','customer','dwdate','supplier')
group by stv_tbl_perm.name
order by 1 asc;
```

#### 查询单表每个字段用的存储大小，单位M
```
select col, max(blocknum) from stv_blocklist b, stv_tbl_perm p where (b.tbl=p.id) and name ='lineorder' and col < 6 group by name, col order by col;
```

#### 查询每个slice上面的数据条数
```
select trim(name) as table, slice, sum(num_values) as rows, min(minvalue), max(maxvalue) from svv_diskusage where name in ('table1', 'table2') and col =0 group by name, slice order by name, slice;
```


#### 测试
更新操作：从原表通过主键 in查一批数据，写到新表，删除原表数据，把新表数据插入到原表，删除新表，整个过程测试一次
##### 单节点性能
写入性能还是不错，执行一条6000条的写入，只需要125ms，一个查询基本还是需要几秒
* 批量更新6000条数据，单线程，大概要25s左右，redshift端执行的时间20s左右
* 批量更新3000条数据，单线程，大概要21s左右，redshift端执行的时间18s左右
* 批量更新500条数据，单线程，大概要20s左右，redshift端执行的时间19s左右
* 批量更新3000条数据，2线程，大概要37s左右，redshift端执行的时间22s左右
* 批量更新3000条数据，4线程，大概要60s左右，redshift端执行的时间36s左右

##### 3节点(低配的计算优化型节点)性能
写入，一条6000条的写入，140ms左右，简单查询在1s以内
* 批量更新6000条数据，单线程，大概要13s左右，redshift端执行的时间5s左右
* 批量更新3000条数据，单线程，大概要9s左右，redshift端执行的时间5s左右
* 批量更新500条数据，单线程，大概要7s左右，redshift端执行的时间5s左右
* 批量更新3000条数据，2线程，大概要17s左右，redshift端执行的时间7s左右
* 批量更新3000条数据，6线程，大概要47s左右，redshift端执行的时间6s左右
* 批量更新3000条数据，12线程，大概要47s左右，redshift端执行的时间18-25s左右

##### 压缩性能
1亿行的数据，原数据默认压缩用了21G硬盘，改用推荐压缩之后6

sql例子：
```
create table if not exists database_name.namespace_name.table_name ( 
id char(32) not null encode raw, 
appr_key char(32) encode lzo, 
aff_id int encode delta32k, 
aff_name varchar(64) encode lzo, 
ad_cost float default 0 encode bytedict, 
session_rt timestamp encode lzo, 
time_diff int default 0 encode delta 
) 
distkey(id) 
sortkey(id, aff_id)
```
