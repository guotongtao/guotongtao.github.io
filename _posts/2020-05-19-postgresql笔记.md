---
layout: post  
title: "postgresql笔记"  
categories: postgresql

tags: postgresql 笔记 



* content  
{:toc}  
# 流复制

## 配置



* 主库：
  yum install postgresql-server
  postgresql-setup initdb
  vi /var/lib/pgsql/data/postgresql.conf 
    wal_level = hot_standby     # write ahead log，流复制时为hot_standby
    max_wal_senders = 4         # 流复制的最大连接数
    wal_keep_segments = 16      # 流复制保留的最大xlog数
    listen_addresses='*'
  vi /var/lib/pgsql/data/pg_hba.conf
    host    replication     repuser        192.168.107.0/24          md5
  systemctl enable postgresql
  systemctl start postgresql
  CREATE USER repuser replication LOGIN CONNECTION LIMIT 3 ENCRYPTED PASSWORD 'Rep_123';











* 备库：
  yum install postgresql-server
  pg_basebackup -F p --progress -D /var/lib/pgsql/data/  -h 192.168.107.218 -p 5432 -U repuser --password
  cp /usr/share/pgsql/recovery.conf.sample /var/lib/pgsql/data/
  vi /var/lib/pgsql/data/postgresql.conf
    hot_standby = on
    hot_standby_feedback = on
  mv  recovery.conf.sample recovery.conf
  vi recovery.conf
    standby_mode = on
    primary_conninfo = 'host=192.168.107.218 port=5432 user=repuser password=Rep_123'
    trigger_file = 'failover.now'  
    recovery_target_timeline = 'latest'

  chown -R postgres:postgres /var/lib/pgsql/data/
  systemctl enable postgresql
  systemctl start postgresql

## 参数说明

max_wal_senders (integer)
指定来自备用服务器或流基础备份客户端的并发连接的最大数目 （即同时运行WAL发送者进程的最大数目）。 默认值是零，这意味着禁用复制。 WAL发送者进程计算连接总数， 因此参数不能高于max_connections。 这个参数只能在服务器启动时设置。wal_level必须设置 为archive或者hot_standby允许来自备用服务器的连接。

wal_keep_segments (integer)
指定在pg_xlog目录下的以往日志文件段的最小数量， 如果备用服务器为了流复制需要获取它们。 那么每个段通常是16兆字节。 如果备用服务器连接到发送服务器落后于wal_keep_segments段，那么发送服务器可能会删除WAL段仍需要待机状态，在这种情况下， 复制连接将被终止。下游连接也将最终失败，因为其结果。（但是，备用服务器可以从归档文件读取的段进行恢复，如果WAL归档在使用中。）

设置保留在pg_xlog中的段最小数量; 该系统可能需要为WAL归档或从检查点恢复保留更多段。 如果wal_keep_segments为0（默认）， 系统不保留备用目的的任何额外段， 所以提供给备用服务器的旧WAL段数是以前检查点定位函数和WAL归档状态信息。 这个参数只能在postgresql.conf文件或服务器命令行上设置。
wal_sender_timeout (integer)
终止比指定毫秒数闲置更长时间的复制连接。 这对于发送服务器检测待机死机或网络中断是很有帮助的。 零值将禁用超时机制。 此参数只能在postgresql.conf文件或服务器命令行上设置。 默认值是60秒。



执行如下命令查看快照，它返回主库记录点、备库记录点；主库每增加一条写入，记录点的值就会加1。
select txid_current_snapshot();



## 查看主备同步状态
主库
postgres=# select * from pg_stat_replication;
  pid  | usesysid | usename | application_name |  client_addr   | client_hostname | client_port |         backend_start                          |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state

-------+----------+---------+------------------+----------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+-----------
-
 12801 |    16384 | replica | walreceiver      | 192.168.162.36 |                 |       36546 | 2019-01-17 11:56:57.754828+08 | streaming | 0/5023B78     | 0/5023B78      | 0/5023B78      | 0/5023B78       |             0 | async
(1 row)


主库上通过执行函数把WAL位置转换成WAL文件名与偏移量
select  *   from pg_xlogfile_name_offset('XXX');

在备库上查看接受的wal日志和wal日志的状态 
select pg_last_xlog_receive_location(),pg_last_xlog_replay_location(),pg_last_xact_replay_timestamp();

postgres=# select pg_current_xlog_insert_location(),pg_current_xlog_location();
 pg_current_xlog_insert_location | pg_current_xlog_location 
---------------------------------+--------------------------
 0/50014F0                       | 0/50014F0
(1 行记录)

postgres=#  select pg_xlogfile_name_offset('0/50014F0');

pg_xlogfile_name_offset     

 (000000010000000000000005,5360)
(1 行记录)

postgres=#  select pg_xlogfile_name('0/50014F0');

​     pg_xlogfile_name     

 000000010000000000000005
(1 行记录)

切换wal日志
postgres=# select pg_switch_xlog();



# 日志

PostgreSQL中有三种日志，pg_log，pg_xlog和pg_clog。分别记录一下，提醒自己。
用处 
pg_log 
该文件夹中的日志一般用来记录服务器与DB的状态，如各种Error信息，定位慢查询SQL，数据库的启动关闭信息，发生checkpoint过于频繁等的告警信息等。linux自带的路径一般在/var/log/postgres下面。该日志有.csv格式和.log。这种日志是可以被清理删除不影响DB的正常运行。
当我们有遇到DB无法启动或者更改参数没有生效时，第一个想到的就是查看这个日志。 

如果服务无法启动，该日志文件夹下的日志没有记录，建议查看操作系统的事件查看器的日志。有助于快速定位问题。第一次有意识去看事件查看器的日志还是翔哥传授，谢谢翔哥哈，虽然我@他他也看不见~~~~

pg_xlog 
该文件夹中的日志是记录的Postgresql的WAL信息，也就是一些事务日志信息(transaction log)，默认单个大小是16M，源码安装的时候可以更改其大小。这些信息通常名字是类似'000000010000000000000013'这样的文件，这些日志会在 定时回滚恢复(PITR)， 流复制(Replication Stream)以及归档时能被用到，这些日志是非常重要的，记录着数据库发生的各种事务信息，不得随意删除或者移动这类日志文件，不然你的数据库会有无法恢复的风险 

当归档或者流复制发生异常的时候，事务日志会不断地生成，有可能会造成磁盘空间被塞满，最终导致DB挂掉或者起不来。遇到这种情况不用慌，可以先关闭归档或者流复制功能，备份pg_xlog日志到其他地方，但请不要删除。然后删除较早时间的的pg_xlog，有一定空间后再试着启动Postgres。 

pg_clog 
pg_clog这个文件也是事务日志文件，但与pg_xlog不同的是它记录的是事务的元数据(metadata)，这个日志告诉我们哪些事务完成了，哪些没有完成。这个日志文件一般非常小，但是重要性也是相当高，不得随意删除或者对其更改信息。

总结：
pg_log记录各种Error信息，以及服务器与DB的状态信息，可由用户随意更新删除
pg_xlog与pg_clog记录数据库的事务信息，不得随意删除更新，做物理备份时要记得备份着两个日志。

# 配置优化
配置文件详解
https://www.cnblogs.com/zhaowenzhong/p/5667434.html

1.shared_buffers
PostgreSQL既使用自身的缓冲区，也使用内核缓冲IO。这意味着数据会在内存中存储两次，首先是存入PostgreSQL缓冲区，然后是内核缓冲区。这被称为双重缓冲区处理。对大多数操作系统来说，这个参数是最有效的用于调优的参数。此参数的作用是设置PostgreSQL中用于缓存的专用内存量。
shared_buffers的默认值设置得非常低，因为某些机器和操作系统不支持使用更高的值。但在大多数现代设备中，通常需要增大此参数的值才能获得最佳性能。
建议的设置值为机器总内存大小的25％，但是也可以根据实际情况尝试设置更低和更高的值。实际值取决于机器的具体配置和工作的数据量大小。举个例子，如果工作数据集可以很容易地放入内存中，那么可以增加shared_buffers的值来包含整个数据库，以便整个工作数据集可以保留在缓存中。
在生产环境中，将shared_buffers设置为较大的值通常可以提供非常好的性能，但应当时刻注意找到平衡点。
设置数据库服务器将使用的共享内存缓冲区数量。缺省通常是128兆字节(128MB)， 但是如果你的内核设置不支持这么大，那么可以少些(在initdb的时候决定)。 每个缓冲区大小的典型值是128千字节，（BLCKSZ的非缺省值改变最小值） 不过，这个数值比最小值大一些通常需要更好的性能。 这个选项只能在服务器启动的时候设置。
如果你有1GB或更多内存的专用数据库服务器， 对于shared_buffers合理的初始值是您的系统内存的25%。 有一些工作负载，甚至在那里对于shared_buffers大设置是有效的， 但因为PostgreSQL也依赖于操作系统缓存， 它是不可能的，RAM到shared_buffers的多于40%的分配比更少数量的工作的更好。 对于shared_buffers的大量设置 通常要求相应增加checkpoint_segments， 为了延长写大量新的或者需较长时间修改的数据的进程。
对于少于1GB RAM系统， 较小百分比内存是相应的， 以便为操作系统留有足够的空间。 此外，在Windows上，shared_buffers大点的值不是很有效。 您可能会发现更好的结果保持设置相对较低，并且使用操作系统的缓存代替。 在Windows系统上shared_buffers的有用范围一般是从64MB到512MB。
查看当前shared_buffers的值：
postgres=# show shared_buffers;

2.wal_buffers
PostgreSQL将其WAL（预写日志）记录写入缓冲区，然后将这些缓冲区刷新到磁盘。由wal_buffers定义的缓冲区的默认大小为16MB，但如果有大量并发连接的话，则设置为一个较高的值可以提供更好的性能。
查看当前wal_buffers的值：
postgres=# show wal_buffers;

3.effective_cache_size
effective_cache_size提供可用于磁盘高速缓存的内存量的估计值。它只是一个建议值，而不是确切分配的内存或缓存大小。它不会实际分配内存，而是会告知优化器内核中可用的缓存量。在一个索引的代价估计中，更高的数值会使得索引扫描更可能被使用，更低的数值会使得顺序扫描更可能被使用。在设置这个参数时，还应该考虑PostgreSQL的共享缓冲区以及将被用于PostgreSQL数据文件的内核磁盘缓冲区。默认值是4GB。
查看当前effective_cache_size的值：
postgres=# show effective_cache_size;

4.work_mem
此配置用于复合排序。内存中的排序比溢出到磁盘的排序快得多，设置非常高的值可能会导致部署环境出现内存瓶颈，因为此参数是按用户排序操作。如果有多个用户尝试执行排序操作，则系统将为所有用户分配大小为work_mem *总排序操作数的空间。全局设置此参数可能会导致内存使用率过高，因此强烈建议在会话级别修改此参数值。默认值为4MB。
查看当前work_mem的值：
postgres=# show work_mem;

5.maintenance_work_mem
maintenance_work_mem是用于维护任务的内存设置。默认值为64MB。设置较大的值对于VACUUM，RESTORE，CREATE INDEX，ADD FOREIGN KEY和ALTER TABLE等操作的性能提升效果显著。
查看当前maintenance_work_mem的值：]
postgres=# show maintenance_work_mem;

6.synchronous_commit
此参数的作用为在向客户端返回成功状态之前，强制提交等待WAL被写入磁盘。这是性能和可靠性之间的权衡。如果应用程序被设计为性能比可靠性更重要，那么关闭synchronous_commit。这意味着成功状态与保证写入磁盘之间会存在时间差。在服务器崩溃的情况下，即使客户端在提交时收到成功消息，数据也可能丢失。
查看当前synchronous_commit的设置值：
postgres=# show synchronous_commit;

7.checkpoint_timeout和checkpoint_completion_target
PostgreSQL将更改写入WAL。检查点进程将数据刷新到数据文件中。发生CHECKPOINT时完成此操作。这是一项开销很大的操作，整个过程涉及大量的磁盘读/写操作。用户可以在需要时随时发出CHECKPOINT指令，或者通过PostgreSQL的参数checkpoint_timeout和checkpoint_completion_target来自动完成。
checkpoint_timeout参数用于设置WAL检查点之间的时间。将此设置得太低会减少崩溃恢复时间，因为更多数据会写入磁盘，但由于每个检查点都会占用系统资源，因此也会损害性能。此参数只能在postgresql.conf文件中或在服务器命令行上设置。
checkpoint_completion_target指定检查点完成的目标，作为检查点之间总时间的一部分。默认值是 0.5。 这个参数只能在postgresql.conf文件中或在服务器命令行上设置。高频率的检查点可能会影响性能。
查看当前checkpoint_timeout和checkpoint_completion_target的值：
postgres=# show checkpoint_timeout;

https://beigang.iteye.com/blog/1199686

# PostgreSQL配置文件--WAL
https://blog.csdn.net/liyingke112/article/details/78805350
# PostgreSQL流复制
http://www.cnblogs.com/lottu/p/7490733.html

# 一些常用命令使用举例

(1)psql加上-E参数，可以把psql中各种以"\"开头的命令执行的实际SQL打印出来
psql -E 
如果你在使用之后，想立即关闭,临时打开就用on
postgres=# \set ECHO_HIDDEN off

(2)显示所有表空间
\db
(3)匹配不同对象类型的\d命令
只显示匹配的表，可以使用\dt命令
只显示索引，可以使用\di命令
只显示序号，可以使用\ds命令
只显示视图，可以使用\dv命令
只显示函数，可以使用\df命令
(4)想显示SQL已执行的时间，可以用\timing命令
(5)显示所有用户或者角色
(6)\x命令-可以把表中的每一行的每列数据都拆分为单行展示
(7)当客户端的字符编码和服务器的不一样时，可能会显示乱码，可以使用\encoding命令来指定客户端的字符编码，如使用\encoding utf8来指定客户端的编码方式为utf8
(8)\dp或\z命令用于显示表的权限分配情况
(9)\pset命令
\pset命令用于指定输出的格式，具体如下：
\pset border 0 : 表示输出内容物边框
\pset border 1 : 表示边框只在内部，默认情况下采用的是该条命令
\pset border 2 : 表示内外都存在边框
(10)\i <SQL文件的路径>
\i <SQL文件的路径>可以在pg中执行外部的SQL语句，这样方便我们执行很复杂的SQL语句。在MySQL中也存在类似的功能，但是实现的方式不一样，在MySQL中执行存储在外部文件中的SQL命令的方式：source <SQL文件的全路径> 或者 \. <SQL文件的全路径>
(11)显示所有模式空间
方法1：test=#  \dnS
        List of schemas
        Name        |  Owner   
--------------------+----------
 information_schema | postgres
 pg_catalog         | postgres
 pg_temp_1          | postgres
 pg_toast           | postgres
 pg_toast_temp_1    | postgres
 public             | postgres
(6 rows)

方法2：test=# select oid,* from pg_catalog.pg_namespace;
  oid  |      nspname       | nspowner |               nspacl                
-------+--------------------+----------+-------------------------------------
    99 | pg_toast           |       10 | 
 11194 | pg_temp_1          |       10 | 
 11195 | pg_toast_temp_1    |       10 | 
    11 | pg_catalog         |       10 | {postgres=UC/postgres,=U/postgres}
  2200 | public             |       10 | {postgres=UC/postgres,=UC/postgres}
 12381 | information_schema |       10 | {postgres=UC/postgres,=U/postgres}
其中information_schema是方便用户查看表／视图／函数信息提供的，它大多是视图，MySQL，SQL Server同样有information_schema这个schema。 
pg_catalog是系统Schema，包含了系统的自带函数／数据类型定义等，pg_catalog是保障postgres正常运转的重要基石。
-- 查看information_schema提供的视图和表
qingping=> select relname, relkind from pg_catalog.pg_class where relnamespace=12921 order by 1;
(12)显示系统表
\dt pg_*
(13)显示系统函数
\dfS
\df *switch*

# 存储结构
postgres数据库目录
show data_directory;

目录结构
base/：存储 database 数据（除了指定其他表空间的）
global/：存储 cluster-wide 表格数据
pg_clog/：存储事务提交的状态数据
pg_commit_ts/：存储事务提交的时间戳数据
pg_dynshmem/：存储动态共享内存子系统的文件
pg_hba.conf：postgresql 配置文件
pg_ident.conf：postgresql 配置文件
pg_logical/：存储 logical decoding 状态数据
pg_multixact/：存储多重事务状态数据的子目录（用于共享的行锁）
pg_notify/：存储 LISTEN/NOTIFY 状态数据
pg_replslot/：存储 replication slot 数据
pg_serial/：存储 committed serializable transactions 信息
pg_snapshots/：存储导出的 snapshots
pg_stat/：存储统计子系统的永久文件
pg_stat_tmp/：存储统计子系统的临时文件
pg_subtrans/：存储子事务状态数据
pg_tblspc/：存储指向表空间的符号链接
pg_twophase/：存储 prepared transactions 的状态文件
PG_VERSION：存储 postgresql 数据库的主版本号
pg_xlog/：存储 WAL (Write Ahead Log) 文件
postgresql.auto.conf：存储由 ALTER SYSTEM 设置的配置
postgresql.conf：postgresql 配置文件
postmaster.opts：存储上一次启动该数据库时用到的命令
postmaster.pid：锁文件，只有在 postgresql 服务运行时存在，存储当前 postmaster 的 PID，PGDATA，postmaster 启动时间，端口号，Unix-domain socket 目录，第一个有效的 listen_address，共享内存的 segment ID


数据文件存储
postgresql 数据库中，每一个对象都对应一个 OID 唯一标识， 而每一个 database 的目录名就存储在与其对应的 OID 目录中： PGDATA/base/oid， 我们可以查询每一个 database 的 OID
SELECT OID,DATNAME FROM PG_DATABASE;
postgres=# SELECT OID,DATNAME FROM PG_DATABASE;
oid | datname 
-------+-----------
1 | template1
13290 | template0
13295 | postgres
16384 | aoldbs
(4 rows)
postgres=# \c aoldbs 连接到库
You are now connected to database "aoldbs" as user "postgres".
aoldbs=# \dt 这个是我的表
List of relations
Schema | Name | Type | Owner 
--------+-----------------------+-------+----------
public | persons | table | postgres
public | sales_detail | table | postgres
public | sales_detail_y2014m01 | table | postgres
public | sales_detail_y2014m02 | table | postgres
public | sales_detail_y2014m03 | table | postgres
public | sales_detail_y2014m04 | table | postgres
public | sales_detail_y2014m05 | table | postgres
public | students | table | postgres
public | test01 | table | postgres
public | testtab01 | table | postgres
这样我就可以看到我的persons表的数据都存放于/opt/postgres/base/16384里边


table 数据存储在哪里？
所有的 table 数据存储在所在数据库的目录里面，table 们是分开存放的， 每一个存储 table 的文件均用 pg_class.relfilenode 命名。两种办法查看具体的位置
aoldbs=# select relfilenode from pg_class where relname ='persons';
AOLDBS
relfilenode
16396
(1 row)
aoldbs=# select pg_relation_filepath('persons');
AOLDBS
pg_relation_filepath 
base/16384/16396
(1 row)

为了避免有些文件系统不支持大文件，postgresql 限制标文件大小不能超过 1G， 因此，当表文件超过 1G 时，会另建一有尾缀文件 relfilenode.1，relfilenode.2， 并以此类推。
当你查看 PGDATA/base/16384/ 目录时， 会发现有些文件命名为 relfilenode_fsm、relfilenode_vm、relfilenode_init， 它们是用来存储与表格相关的 free space map 、visibility map 和 unlogged table 的 initialization fork。
在大多数情况下，表中数据的存放是无序的，我们称之为堆表，heap table.
多条数据一起存放在一个page中，多个page形成一个数据文件，pg中最小的io单位为page，所有每个文件的大小一定是page size的整数倍。

# PostgreSQL表空间、数据库、模式、表、用户/角色之间的关系
（1）角色与用户的关系
    在PostgreSQL中，存在两个容易混淆的概念：角色/用户。之所以说这两个概念容易混淆，是因为对于PostgreSQL来说，这是完全相同的两个对象。唯一的区别是在创建的时候：
 1.我用下面的psql创建了角色kanon：
   CREATE ROLE kanon PASSWORD 'kanon';
   接着我使用新创建的角色kanon登录，PostgreSQL给出拒绝信息：

   FATAL： role 'kanon' is not permitted to log in.
   说明该角色没有登录权限，系统拒绝其登录。 
 2.我又使用下面的psql创建了用户kanon2:
   CREATE USER kanon PASSWORD 'kanon2';
   接着我使用kanon2登录，登录成功。
   难道这两者有区别吗？查看文档，又这么一段说明："CREATE USER is the same as CREATE ROLE except that it implies LOGIN."----CREATE USER除了默认具有LOGIN权限之外，其他与CREATE ROLE是完全相同的。
   为了验证这句话，修改kanon的权限，增加LOGIN权限：ALTER ROLE kanon LOGIN;再次用kanon登录，成功！
   那么，事情就明了了：CREATE ROLE kanon PASSWORD 'kanon' LOGIN 等同于CREATE USER kanon PASSWORD 'kanon'.
   这就是ROLE/USER的区别。

(2）数据库与模式的关系
    看文档了解到：模式（schema）是对数据库（database）逻辑分割。
在数据库创建的同时，就已经默认为数据库创建了一个模式--public，这也是该数据库的默认模式。所有为此数据库创建的对象（表、函数、试图、索引、序列等）都是常见在这个模式中的。
 实验如下：
 1.创建一个数据库dbtt----CREATE DATABASE dbtt;
 2.用kanon角色登录到dbtt数据库,查看dbtt数据库中的所有模式：/dn; 显示结果是只有public一个模式。
 3.创建一张测试表----CREATE TABLE test(id integer not null);
 4.查看当前数据库的列表： /d; 显示结果是表test属于模式public.也就是test表被默认创建在了public模式中。
 5.创建一个新模式kanon，对应于登录用户kanon：CREATE SCHEMA kanon OWNER kanon；
 6.再次创建一张test表，这次这张表要指明模式----CREATE TABLE kanon.test (id integer not null);
 7.查看当前数据库的列表： /d; 显示结果是表test属于模式kanon.也就是这个test表被创建在了kanon模式中。
   得出结论是：数据库是被模式(schema)来切分的，一个数据库至少有一个模式，所有数据库内部的对象(object)是被创建于模式的。用户登录到系统，连接到一个数据库后，是通过该数据库的search_path来寻找schema的搜索顺序,可以通过命令SHOW search_path；具体的顺序，也可以通过SET search_path TO 'schema_name'来修改顺序。
   官方建议是这样的：在管理员创建一个具体数据库后，应该为所有可以连接到该数据库的用户分别创建一个与用户名相同的模式，然后，将search_path设置为"$user"，
   这样，任何当某个用户连接上来后，会默认将查找或者定义的对象都定位到与之同名的模式中。这是一个好的设计架构。

（3）表空间与数据库的关系
    数据库创建语句CREATE DATABASE dbname 默认的数据库所有者是当前创建数据库的角色，默认的表空间是系统的默认表空间--pg_default。
    为什么是这样的呢？因为在PostgreSQL中，数据的创建是通过克隆数据库模板来实现的，这与SQL SERVER是同样的机制。
    由于CREATE DATABASE dbname并没有指明数据库模板，所以系统将默认克隆template1数据库，得到新的数据库dbname。（By default, the new database will be created by cloning the standard system database template1）.

    而template1数据库的默认表空间是pg_default，这个表空间是在数据库初始化时创建的，所以所有template1中的对象将被同步克隆到新的数据库中。
    相对完整的语法应该是这样的：CREATE DATABASE dbname OWNER kanon TEMPLATE template1 TABLESPACE tablespacename; 
    下面我们来做个实验验证一下：
 1.连接到template1数据库，创建一个表作为标记：CREATE TABLE tbl_flag(id integer not null);向表中插入数据INSERT INTO tbl_flag VALUES (1);
 2.创建一个表空间:CREATE TABLESPACE tskanon OWNER kanon LOCATION '/tmp/data/tskanon';在此之前应该确保目录/tmp/data/tskanon存在，并且目录为空。
 3.创建一个数据库，指明该数据库的表空间是刚刚创建的tskanon：CREATE DATABASE dbkanon TEMPLATE template1 OWNERE kanon TABLESPACE tskanon;
 4.查看系统中所有数据库的信息：/l；可以发现，dbkanon数据库的表空间是tskanon,拥有者是kanon;
 5.连接到dbkanon数据库，查看所有表结构:/d;可以发现，在刚创建的数据库中居然有了一个表tbl_flag,查看该表数据，输出结果一行一列，其值为1，说明，该数据库的确是从template1克隆而来。
 仔细分析后，不难得出结论：在PostgreSQL中，表空间是一个目录，里面存储的是它所包含的数据库的各种物理文件。

最后，我们回头来总结一下这张关系网
    表空间是一个存储区域，在一个表空间中可以存储多个数据库，尽管PostgreSQL不建议这么做，但我们这么做完全可行。
    一个数据库并不知直接存储表结构等对象的，而是在数据库中逻辑创建了至少一个模式，在模式中创建了表等对象，将不同的模式指派该不同的角色，可以实现权限分离，又可以通过授权，实现模式间对象的共享，并且，还有一个特点就是：public模式可以存储大家都需要访问的对象。
    这样，我们的网就形成了。可是，既然一个表在创建的时候可以指定表空间，那么，是否可以给一个表指定它所在的数据库表空间之外的表空间呢？
    答案是肯定的!这么做完全可以：那这不是违背了表属于模式，而模式属于数据库，数据库最终存在于指定表空间这个网的模型了吗？！
    是的，看上去这确实是不合常理的，但这么做又是有它的道理的，而且现实中，我们往往需要这么做：将表的数据存在一个较慢的磁盘上的表空间，而将表的索引存在于一个快速的磁盘上的表空间。
    但我们再查看表所属的模式还是没变的，它依然属于指定的模式。所以这并不违反常理。实际上，PostgreSQL并没有限制一张表必须属于某个特定的表空间，我们之所以会这么认为，是因为在关系递进时，偷换了一个概念：模式是逻辑存在的，它不受表空间的限制。
十、系统管理函数
配置设置函数
名字	返回类型	描述
current_setting(setting_name)	text	当前的设置值
set_config(setting_name, new_value, is_local)	text	设置参数并返回新值
current_setting 用于以查询形式获取 setting_name 设置的当前值。它和 SQL 命令 SHOW 是等效的。比如：
SELECT current_setting('datestyle');
* current_setting

 ISO, MDY
(1 row)
set_config 将参数 setting_name 设置为 new_value 。如果 is_local 为 true ，那么新值将只应用于当前事务。如果你希望新值应用于当前会话，那么应该使用 false 。它等效于 SQL 命令 SET 。比如：

SELECT set_config('log_statement_stats', 'off', false);
* set_config

 off
(1 row)

服务器信号函数
名字	返回类型	描述
pg_cancel_backend(pid int)	boolean	取消一个后端的当前查询
pg_reload_conf()	boolean	导致所有服务器进程重新装载它们的配置文件
pg_rotate_logfile()	boolean	滚动服务器的日志文件
如果成功，这些函数返回 true ，否则返回 false 。
pg_cancel_backend 向由 pid 标识的后端进程发送一个查询取消(SIGINT)信号。一个活动的后端进程的 PID 可以从 pg_stat_activity 视图的 procpid 字段找到，或者在服务器上用 ps 列出 postgres 进程。
pg_reload_conf 给服务器发送一个 SIGHUP 信号，导致所有服务器进程重新装载配置文件。
pg_rotate_logfile 给日志文件管理器发送信号，告诉它立即切换到一个新的输出文件。这个函数只有在 redirect_stderr 用于日志输出的时候才有用，否则根本不存在日志文件管理器子进程。

备份控制函数
pg_start_backup(label text)	text	开始执行在线备份
pg_stop_backup()	text	完成执行在线备份
pg_switch_xlog()	text	切换到一个新的事务日志文件
pg_current_xlog_location()	text	获取当前事务日志的写入位置
pg_current_xlog_insert_location()	text	获取当前事务日志的插入位置
pg_xlogfile_name_offset(location text)	text, integer	将事务日志的位置字符串转换为文件名并返回在文件中的字节偏移量
pg_xlogfile_name(location text)	text	将事务日志的位置字符串转换为文件名
pg_start_backup 接受一个用户定义的备份标签(通常这是备份转储文件存放地点的名字)。这个函数向数据库集群的数据目录写入一个备份标签文件，然后以文本方式返回备份的事务日志起始位置。用户不需要关心这个返回值，提供它只是为了万一需要的场合。
postgres=# select pg_start_backup('label_goes_here');
* pg_start_backup

 0/D4445B8
(1 row)
pg_stop_backup 删除 pg_start_backup 创建的标签文件，并且在事务日志归档区里创建一个备份历史文件。这个历史文件包含给予 pg_start_backup 的标签、备份的事务日志起始与终止位置、备份的起始和终止时间。返回值是备份的事务日志终止位置(同样也可能没有什么用)。计算出中止位置后，当前事务日志的插入点将自动前进到下一个事务日志文件，这样，结束的事务日志文件可以被立即归档从而完成备份。
pg_switch_xlog 移动到下一个事务日志文件，以允许将当前日志文件归档(假定你使用连续归档)。返回值是刚刚完成的事务日志文件的事务日志结束位置。如果自从最后一次事务日志切换以来没有活动的事务日志，那么 pg_switch_xlog 什么事也不做，直接返回前一个事务日志文件的结束位置。
pg_current_xlog_location 使用与前面那些函数相同的格式显示当前事务日志的写入位置。类似的，pg_current_xlog_insert_location 显示当前事务日志的插入位置。插入点是事务日志在某个瞬间的"逻辑终点"，而实际的写入位置则是从服务器内部缓冲区写出时的终点。写入位置是可以从服务器外部检测到的终点，如果想归档部分完成的事务日志文件，那么这个通常就是你想要的结果。插入点主要用于服务器调试目的。上述两个函数既是只读操作也不需要超级用户权限。
可以使用 pg_xlogfile_name_offset 从前述函数的返回结果中抽取相应的事务日志文件名称和字节偏移量。例如：
postgres=# select * from pg_xlogfile_name_offset(pg_stop_backup());
        file_name         | file_offset 
--------------------------+-------------
 00000001000000000000000D |     4039624
(1 row)
类似的，pg_xlogfile_name 仅仅抽取事务日志文件名称。如果给定的事务日志位置恰好位于事务日志文件的交界上，这两个函数都返回前一个事务日志文件的名字。这对于管理事务日志归档来说通常是期望的行为，因为前一个文件是当前最后一个需要归档的文件。
postgres=# select pg_current_xlog_insert_location(),pg_current_xlog_location();
 pg_current_xlog_insert_location | pg_current_xlog_location 
---------------------------------+--------------------------
 0/50014F0                       | 0/50014F0
(1 行记录)

postgres=#  select pg_xlogfile_name_offset('0/50014F0');
* pg_xlogfile_name_offset     

 (000000010000000000000005,5360)
(1 行记录)

postgres=#  select pg_xlogfile_name('0/50014F0');
* pg_xlogfile_name     

 000000010000000000000005
(1 行记录)
数据库对象尺寸函数
名字	返回类型	描述
pg_column_size(any)	int	存储一个指定的数值需要的字节数(可能压缩过)
pg_database_size(oid)	bigint	指定 OID 代表的数据库使用的磁盘空间
pg_database_size(name)	bigint	指定名称的数据库使用的磁盘空间
pg_relation_size(oid)	bigint	指定 OID 代表的表或者索引所使用的磁盘空间
pg_relation_size(text)	bigint	指定名称的表或者索引使用的磁盘空间。表名字可以用模式名修饰。
pg_size_pretty(bigint)	text	把字节计算的尺寸转换成一个人类易读的尺寸。
pg_tablespace_size(oid)	bigint	指定 OID 代表的表空间使用的磁盘空间
pg_tablespace_size(name)	bigint	指定名字的表空间使用的磁盘空间
pg_total_relation_size(oid)	bigint	指定 OID 代表的表使用的磁盘空间，包括索引和压缩数据。
pg_total_relation_size(text)	bigint	指定名字的表所使用的全部磁盘空间，包括索引和压缩数据。表名字可以用模式名修饰。
pg_column_size 显示用于存储某个独立数据值的空间。
pg_database_size 和 pg_tablespace_size 接受一个数据库的 OID 或者名字，然后返回该对象使用的全部磁盘空间。
pg_relation_size 接受一个表、索引、压缩表的 OID 或者名字，然后返回它们以字节计的尺寸。
pg_size_pretty 用于把其它函数的结果格式化成一种人类易读的格式，可以根据情况使用KB 、MB 、GB 、TB 。
pg_total_relation_size 接受一个表或者一个压缩表的 OID 或者名称，然后返回以字节计的数据和所有相关的索引和压缩表的尺寸。

通用文件访问函数
名字	返回类型	描述
pg_ls_dir(dirname text)	setof text	列出目录中的文件
pg_read_file(filename text, offset bigint, length bigint)	text	返回一个文本文件的内容
pg_stat_file(filename text)	record	返回一个文件的信息
pg_ls_dir 返回指定目录里面的除了特殊项 "." 和 ".." 之外所有名字。
pg_read_file 返回一个文本文件的一部分，从 offset 开始，返回最多 length 字节(如果先达到文件结尾，则小于这个数值)。如果 offset 是负数，那么它就是相对于文件结尾回退的长度。
pg_stat_file 返回一条记录，其中包含：文件大小、最后访问时间戳、最后更改时间戳、最后文件状态修改时间戳(只在 Unix 平台上可用)、文件创建时间戳(只在 Windows 平台上可用)、是否为目录的 boolean 值。典型的用法：
SELECT * FROM pg_stat_file('filename');
SELECT (pg_stat_file('filename')).modification;

咨询锁函数
名字	返回类型	描述
pg_advisory_lock(key bigint)	void	获取排它咨询锁
pg_advisory_lock(key1 int, key2 int)	void	获取排它咨询锁
pg_advisory_lock_shared(key bigint)	void	获取共享咨询锁
pg_advisory_lock_shared(key1 int, key2 int)	void	获取共享咨询锁
pg_try_advisory_lock(key bigint)	boolean	尝试获取排它咨询锁
pg_try_advisory_lock(key1 int, key2 int)	boolean	尝试获取排它咨询锁
pg_try_advisory_lock_shared(key bigint)	boolean	尝试获取共享咨询锁
pg_try_advisory_lock_shared(key1 int, key2 int)	boolean	尝试获取共享咨询锁
pg_advisory_unlock(key bigint)	boolean	释放排它咨询锁
pg_advisory_unlock(key1 int, key2 int)	boolean	释放排它咨询锁
pg_advisory_unlock_shared(key bigint)	boolean	释放共享咨询锁
pg_advisory_unlock_shared(key1 int, key2 int)	boolean	释放共享咨询锁
pg_advisory_unlock_all()	void	释放当前会话持有的所有咨询锁
pg_advisory_lock 锁定一个应用程序定义的资源，该资源可以用一个 64 位或两个不重叠的 32 位键值标识。如果已经有另外的会话锁定了该资源，那么该函数将会阻塞到该资源可用为止。这个锁是排它的。多个锁定请求将会被压入栈中，因此，如果同一个资源被锁定了三次，那么它必须被解锁三次以将资源释放给其它会话使用。
pg_advisory_lock_shared 类似于 pg_advisory_lock ，不同之处仅在于共享锁可以和其它请求共享锁的会话共享，但排它锁除外。
pg_try_advisory_lock 类似于 pg_advisory_lock ，不同之处在于该函数不会阻塞以等待资源的释放。它要么立即获得锁并返回 true ，要么返回 false 表示目前不能锁定。
pg_try_advisory_lock_shared 类似于 pg_try_advisory_lock ，不同之处在于该函数尝试获得共享锁而不是排它锁。
pg_advisory_unlock 释放先前取得的排它咨询锁。如果释放成功则返回 true 。如果指定的锁实际上并未持有，那么它将返回 false 并在服务器中产生一条 SQL 警告信息。
pg_advisory_unlock_shared 类似于 pg_advisory_unlock 不同之处在于该函数释放的是共享咨询锁。
pg_advisory_unlock_all 将会释放当前会话持有的所有咨询锁，该函数在会话结束的时候被隐含调用，即使客户端异常地断开连接也是一样。

# 系统视图
pg_stat_replication 一监控Streaming Replication集群



# 备份与恢复
Postgresql的三种备份方式
https://toplchx.iteye.com/blog/2093821
https://blog.csdn.net/international24/article/details/82689136
备份与恢复
https://www.yiibai.com/manual/postgresql/continuous-archiving.html
rman备份
https://www.cnblogs.com/lottu/p/7490615.html
连续归档和时间点恢复（PITR）
http://www.postgres.cn/docs/9.6/continuous-archiving.html

# 深入理解 template1 和 template0
https://blog.csdn.net/baidu_33387365/article/details/80883142

# 当前事务日志的插入位置 和当前日志写入位置的区别？
pg_current_xlog_insert_location() text 获取当前事务日志的插入位置 ，指还在wal buffer中的位置。
pg_current_xlog_location() text 获取当前事务日志的写入位置，指已经调用了write后的位置（但是sync到磁盘之前），所以一定不在wal buffer了.
查看具体的xlog文件

# postgresql wal
postgresql wal 解释
https://blog.csdn.net/cdnight/article/details/79455366
Postgresql之WAL log机制
https://blog.csdn.net/Linzhongyilisha/article/details/79758593
PostgreSQL持久性优化机制-简书
https://www.jianshu.com/p/a37ceed648a8
postgresql wal日志参数浅析
http://itindex.net/detail/50712-postgresql-wal-%E6%97%A5%E5%BF%97
认识PostgreSQL WAL
https://www.jianshu.com/p/872a2ef931b5

# PostgreSQL流复制参数max_wal_senders详解
https://www.cnblogs.com/weiji100/p/3610602.html

# PostgreSQL的WAL日志解析工具pg_waldump 浅谈
https://blog.csdn.net/yaoqiancuo3276/article/details/80826073

#  PostgreSQL 逻辑结构和权限体系介绍
https://www.cnblogs.com/jianyungsun/p/7591753.html

#  postgres&oracle备份恢复效率比较
http://blog.sina.com.cn/s/blog_544a710b0101a7tb.html

# pg_restore及psql恢复数据的用法
https://blog.csdn.net/lk_db/article/details/77971634

# postgresql查看用户连接以及杀死连接的会话
https://blog.csdn.net/db_su/article/details/78204101

# pg_restore使用
https://my.oschina.net/yafeishi/blog/742307

# postgresql用户模式权限介绍
https://blog.csdn.net/su377486/article/details/80672867

# pg_basebackup 命令行参数
https://blog.csdn.net/feixiangtianshi/article/details/49152691

# 开启归档
步骤一：修改postgresql的配置文件(postgresql.conf)
           wal_level=hot_standby
           archive_mode =on 
           archive_command ='test ! -f /storage/pgsql/backups/%f && cp %p /storage/pgsql/backups/archive/%f'
           archive_command ='DATE=`date +%Y%m%d`;DIR="/home/postgres/arch/$DATE";(test -d $DIR || mkdir -p $DIR)&& cp %p $DIR/%f'
           ps：%p 是指相对路径  %f是指文件名
步骤二：创建归档路径
          mkdir  -p /home/postgres/arch
          chown -R postgres:postgres /home/postgres/arch
步骤三：重启数据库
步骤四：验证归档是否正常
          postgres=# checkpoint;
                            CHECKPOINT
           postgres=# select pg_switch_xlog();
             pg_switch_xlog 
             ----------------
​             1/760000E8
​             (1 row)
​          postgres@ubuntu:~$ cd /home/postgres/data/data_1999/arch/
​          postgres@ubuntu:~/data/data_1999/arch$ ls
​            20150603
​          postgres@ubuntu:~/data/data_1999/arch$ cd 20150603/
​           postgres@ubuntu:~/data/data_1999/arch/20150603$ ls
​           000000010000000100000074  000000010000000100000075  000000010000000100000076

# PostgreSQL日志号LSN和wal日志文件简记
https://www.cnblogs.com/kuang17/p/8758622.html

# PostgreSQL 使用pg_xlogdump找到误操作事务号
https://yq.aliyun.com/articles/71

# pg_waldump pg_xlogdump 的初步使用
https://blog.csdn.net/ctypyb2002/article/details/80640942

# how to obtain a list of restore points?
https://dba.stackexchange.com/questions/118122/how-to-obtain-a-list-of-restore-points

# Postgresql更新已存在数据库下表的拥有者
https://www.oschina.net/code/snippet_2369011_48647

# 如何用好PostgreSQL的备份与恢复？
https://baijiahao.baidu.com/s?id=1582080498650348658

