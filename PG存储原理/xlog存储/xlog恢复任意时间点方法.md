```
一般用户在发生误操作后，可能会非常模糊的记得一个大概操作的时间点，那么通过这个线索，我们怎么能定位到用户误操作的精准事务号呢？
从而使用PITR恢复到发生误操作前的数据库状态。
我们可以使用审计日志，找到精准的时间，假设你开了所有SQL的审计日志(log_statement = 'all')。
然后找到精准的XID，需要使用pg_xlogdump。
一个例子：
用户在某个时间点2015-09-21 10:10:00 +08发生了一笔误操作，删除了某个表tbl的一批数据。
首先我们可以在日志中，根据用户给的模糊时间，找到精准的时间。

将数据库恢复到这之前的一个时间（最好是给出时间点的上一个检查点之前的时间），并停止恢复。
假设检查点时5分钟会做一次的，那么我们可以选择5分钟前的一个时间点。
这么做是为了防止PITR到误操作后，那就白搞了。

$vi $PGDATA/recovery.conf 
recovery_target_inclusive = false
restore_command = 'cp /tmp/%f %p'
recovery_target_time = '2015-09-21 10:00:00 +08'   # 选择了一个10分钟前的时间
standby_mode = on
pause_at_recovery_target = true

数据库起来之后，去查找tbl对应的filenode，要用它来分析pg_xlog
digoal=# select oid from pg_class where relname='tbl';
   oid    
----------
 34874054e
(1 row)
digoal=# select pg_relation_filenode(34874054);
 pg_relation_filenode 
----------------------
             37201015
(1 row)
digoal=# select pg_relation_filepath(34874054);
  pg_relation_filepath  
------------------------
 base/34873862/37201015
(1 row)

然后下载这个误操作前的基础备份，以及到误操作时产生的所有XLOG。
000000040000006500000095
000000040000006500000096
000000040000006500000097

使用pg_xlogdump分析XLOG。
$pg_xlogdump -b 000000040000006500000097 000000040000006500000097 | less
找到误操作的XID了，就是1485021，因为这笔事务日志包含了大量的tbl表的delete，而且上一个事务结束的时间点2015-09-21 10:13:22.038262和用户给的误操作时间点吻合。
rmgr: Heap        len (rec/tot):     50/    82, tx:    1485020, lsn: 65/9760DDF8, prev 65/9760DDC0, bkp: 0000, desc: hot_update: rel 1663/13003/16387; tid 0/36 xmax 1485020 ; new tid 0/37 xmax 0
rmgr: Transaction len (rec/tot):     12/    44, tx:    1485020, lsn: 65/9760DE50, prev 65/9760DDF8, bkp: 0000, desc: commit: 2015-09-21 10:13:22.038262 CST

rmgr: Heap        len (rec/tot):     26/  8218, tx:    1485021, lsn: 65/9760DE80, prev 65/9760DE50, bkp: 1000, desc: delete: rel 1663/34873862/37201015; tid 0/1 KEYS_UPDATED 
        backup bkp #0; rel 1663/34873862/37201015; fork: main; block: 0; hole: offset: 184, length: 56
rmgr: Heap        len (rec/tot):     26/    58, tx:    1485021, lsn: 65/9760FEB8, prev 65/9760DE80, bkp: 0000, desc: delete: rel 1663/34873862/37201015; tid 0/2 KEYS_UPDATED 
rmgr: Heap        len (rec/tot):     26/    58, tx:    1485021, lsn: 65/9760FEF8, prev 65/9760FEB8, bkp: 0000, desc: delete: rel 1663/34873862/37201015; tid 0/3 KEYS_UPDATED 
rmgr: Heap        len (rec/tot):     26/    58, tx:    1485021, lsn: 65/9760FF38, prev 65/9760FEF8, bkp: 0000, desc: delete: rel 1663/34873862/37201015; tid 0/4 KEYS_UPDATED 
rmgr: Heap        len (rec/tot):     26/    58, tx:    1485021, lsn: 65/9760FF78, prev 65/9760FF38, bkp: 0000, desc: delete: rel 1663/34873862/37201015; tid 0/5 KEYS_UPDATED 
.....

解释一下
desc: delete: rel 1663/34873862/37201015;
1663表示表空间的oid
34873862对应数据库的oid
37201015对应table的filenode
contrib/pg_xlogdump/pg_xlogdump.c
                        printf("\tbackup bkp #%u; rel %u/%u/%u; fork: %s; block: %u; hole: offset: %u, length: %u\n",
                                   bkpnum,
                                   bkpb.node.spcNode, bkpb.node.dbNode, bkpb.node.relNode,
                                   forkNames[bkpb.fork],
                                   bkpb.block, bkpb.hole_offset, bkpb.hole_length);
src/include/storage/relfilenode.h
typedef struct RelFileNode
{
        Oid                     spcNode;                /* tablespace */
        Oid                     dbNode;                 /* database */
        Oid                     relNode;                /* relation */
} RelFileNode;
src/include/access/xlog_internal.h
typedef struct BkpBlock
{
        RelFileNode node;                       /* relation containing block */
        ForkNumber      fork;                   /* fork within the relation */
        BlockNumber block;                      /* block number */
        uint16          hole_offset;    /* number of bytes before "hole" */
        uint16          hole_length;    /* number of bytes in "hole" */

        /* ACTUAL BLOCK DATA FOLLOWS AT END OF STRUCT */
} BkpBlock;

现在已经知道发生误操作前的最后一笔结束事务的XID，需要把recovery.conf改为xid恢复。
recovery_target_inclusive = true
restore_command = 'cp /tmp/%f %p'
recovery_target_xid = '1485020'
standby_mode = on
pause_at_recovery_target = true
重启数据库，现在已经恢复到真正的目标了。
可以导出tbl表，恢复被删除的数据。

[参考]
1. http://blog.163.com/digoal@126/blog/static/163877040201303082942271/
2. http://blog.163.com/digoal@126/blog/static/16387704020155199222755/
```