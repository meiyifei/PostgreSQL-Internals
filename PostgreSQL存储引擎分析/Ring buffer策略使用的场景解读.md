# 		Ring buffer策略使用场景解读

参考资料:<https://yq.aliyun.com/articles/697694> 

### 一、Ring buffer介绍

```bash
#src/backend/storage/buffer/README
Buffer Ring Replacement Strategy
---------------------------------

When running a query that needs to access a large number of pages just once,
such as VACUUM or a large sequential scan, a different strategy is used.
A page that has been touched only by such a scan is unlikely to be needed
again soon, so instead of running the normal clock sweep algorithm and
blowing out the entire buffer cache, a small ring of buffers is allocated
using the normal clock sweep algorithm and those buffers are reused for the
whole scan.  This also implies that much of the write traffic caused by such
a statement will be done by the backend itself and not pushed off onto other
processes.

当运行一个只需要访问一次大量页面的查询时，例如真空或大的顺序扫描，则使用不同的策略。仅被这种扫描触碰过的页面不太可能很快再次需要，因此不必运行普通的时钟扫描算法并清空整个缓冲区缓存，使用普通的时钟扫描算法分配一小圈缓冲区，这些缓冲区在整个扫描过程中被重用。这也意味着，由这样一个语句引起的大部分写通信量将由后端本身完成，而不是推送到其他进程上。

For sequential scans, a 256KB ring is used. That's small enough to fit in L2
cache, which makes transferring pages from OS cache to shared buffer cache
efficient.  Even less would often be enough, but the ring must be big enough
to accommodate all pages in the scan that are pinned concurrently.  256KB
should also be enough to leave a small cache trail for other backends to
join in a synchronized seq scan.  If a ring buffer is dirtied and its LSN
updated, we would normally have to write and flush WAL before we could
re-use the buffer; in this case we instead discard the buffer from the ring
and (later) choose a replacement using the normal clock-sweep algorithm.
Hence this strategy works best for scans that are read-only (or at worst
update hint bits).  In a scan that modifies every page in the scan, like a
bulk UPDATE or DELETE, the buffers in the ring will always be dirtied and
the ring strategy effectively degrades to the normal strategy.

对于顺序扫描，使用256KB Ring Buffer。小到可以放在L2缓存中，使页面从操作系统缓存传输到共享缓冲区缓存效率高。即使少一点就足够了，但buffer必须足够大以容纳扫描中同时固定的所有页。256KB也应该足够为其他后端留下一个小的缓存跟踪加入同步序列扫描。如果一个环缓冲区被污染并且它的LSN更新后，我们通常要先写和刷新WAL重新使用缓冲区；在这种情况下，我们将丢弃环中的缓冲区，然后（稍后）使用常规时钟扫描算法选择替换。因此，此策略最适合只读（或最坏情况下）的扫描更新提示位）。在扫描中修改扫描中的每一页，如批量更新或删除时，环中的缓冲区将始终被弄脏环策略有效地退化为正常策略。

VACUUM uses a 256KB ring like sequential scans, but dirty pages are not
removed from the ring.  Instead, WAL is flushed if needed to allow reuse of
the buffers.  Before introducing the buffer ring strategy in 8.3, VACUUM's
buffers were sent to the freelist, which was effectively a buffer ring of 1
buffer, resulting in excessive WAL flushing.  Allowing VACUUM to update
256KB between WAL flushes should be more efficient.

VACUUM使用一个256KB的环就像形顺序扫描一样，但脏页不会从环中删除。相反，如果需要允许重用缓冲区，则刷新WAL。在8.3中引入缓冲环策略之前，VACUUM的缓冲区被发送到freelist，freelist实际上是一个1缓冲区的缓冲环，导致WAL刷新过多。允许真空在WAL刷新之间更新256KB应该更有效。

Bulk writes work similarly to VACUUM.  Currently this applies only to
COPY IN and CREATE TABLE AS SELECT.  (Might it be interesting to make
seqscan UPDATE and DELETE use the bulkwrite strategy?)  For bulk writes
we use a ring size of 16MB (but not more than 1/8th of shared_buffers).
Smaller sizes have been shown to result in the COPY blocking too often
for WAL flushes.  While it's okay for a background vacuum to be slowed by
doing its own WAL flushing, we'd prefer that COPY not be subject to that,
so we let it use up a bit more of the buffer arena.

批量写入的工作方式与VACUUM类似。目前，这仅适用于COPY IN 与 CREATE TABLE AS SELECT。（让seqscan更新和删除使用bulkwrite策略是否有趣？）对于大容量写入，我们使用16MB的环大小（但不超过共享缓冲区的1/8）。较小的大小已被显示为在COPY时会导致WAL刷新的阻塞太频繁。虽然background vacuum可以通过自己的WAL flushing来减缓速度，但我们希望该COPY不受此影响，所以我们让它占用更多的缓冲区。
```

​	在读写大表时，PostgreSQL将使用Ring buffer而不是Shared Buffer。ring buffer就是在当需要大量访问页面的情况下如vacuum或者大量的全表扫描时采用的一种特殊的策略。不会像正常的时钟扫描算法交换出整个缓冲区，而是在一小块缓冲区上使用时钟扫描算法，并会重用缓冲区完成整个扫描。这样就会避免大量全表扫描带来的缓冲区命中率的下降。

### 二、Ring buffer策略使用场景

什么时候会使用到Ring buffer呢？

```c
#src/backend/storage/buffer/freelist.c

/* ----------------------------------------------------------------
 *				Backend-private buffer ring management
 * ----------------------------------------------------------------
 */


/*
 * GetAccessStrategy -- create a BufferAccessStrategy object
 *
 * The object is allocated in the current memory context.
 */
BufferAccessStrategy
GetAccessStrategy(BufferAccessStrategyType btype)
{
	BufferAccessStrategy strategy;
	int			ring_size;

	/*
	 * Select ring size to use.  See buffer/README for rationales.
	 *
	 * Note: if you change the ring size for BAS_BULKREAD, see also
	 * SYNC_SCAN_REPORT_INTERVAL in access/heap/syncscan.c.
	 */
	switch (btype)
	{
		case BAS_NORMAL:
			/* if someone asks for NORMAL, just give 'em a "default" object */
			return NULL;

		case BAS_BULKREAD:
			ring_size = 256 * 1024 / BLCKSZ;
			break;
		case BAS_BULKWRITE:
			ring_size = 16 * 1024 * 1024 / BLCKSZ;
			break;
		case BAS_VACUUM:
			ring_size = 256 * 1024 / BLCKSZ;
			break;

		default:
			elog(ERROR, "unrecognized buffer access strategy: %d",
				 (int) btype);
			return NULL;		/* keep compiler quiet */
	}

	/* Make sure ring isn't an undue fraction of shared buffers */
	ring_size = Min(NBuffers / 8, ring_size);

	/* Allocate the object and initialize all elements to zeroes */
	strategy = (BufferAccessStrategy)
		palloc0(offsetof(BufferAccessStrategyData, buffers) +
				ring_size * sizeof(Buffer));

	/* Set fields that don't start out zero */
	strategy->btype = btype;
	strategy->ring_size = ring_size;

	return strategy;
}
```

缓冲区访问策略类型

```c
src/include/storage/bufmgr.h
/* Possible arguments for GetAccessStrategy() */
typedef enum BufferAccessStrategyType
{
	BAS_NORMAL,					/* Normal random access */
	BAS_BULKREAD,				/* Large read-only scan (hint bit updates are
								 * ok) */
	BAS_BULKWRITE,				/* Large multi-block write (e.g. COPY IN) */
	BAS_VACUUM					/* VACUUM */
} BufferAccessStrategyType;
```

- BAS_BULKREAD

  批量读会使用Ring buffer，那么在PostgreSQL中怎样才算批量读呢？

  ```c
  #src/backend/access/heap/heapam.c
  
  /*
  	 * If the table is large relative to NBuffers, use a bulk-read access
  	 * strategy and enable synchronized scanning (see syncscan.c).  Although
  	 * the thresholds for these features could be different, we make them the
  	 * same so that there are only two behaviors to tune rather than four.
  	 * (However, some callers need to be able to disable one or both of these
  	 * behaviors, independently of the size of the table; also there is a GUC
  	 * variable that can disable synchronized scanning.)
  	 *
  	 * Note that table_block_parallelscan_initialize has a very similar test;
  	 * if you change this, consider changing that one, too.
  	 */
  
  	if (!RelationUsesLocalBuffers(scan->rs_base.rs_rd) &&
  		scan->rs_nblocks > NBuffers / 4)
  	{
  		allow_strat = (scan->rs_base.rs_flags & SO_ALLOW_STRAT) != 0;
  		allow_sync = (scan->rs_base.rs_flags & SO_ALLOW_SYNC) != 0;
  	}
  	else
  		allow_strat = allow_sync = false;
  
  	if (allow_strat)
  	{
  		/* During a rescan, keep the previous strategy object. */
  		if (scan->rs_strategy == NULL)
  			scan->rs_strategy = GetAccessStrategy(BAS_BULKREAD);
  	}
  	else
  	{
  		if (scan->rs_strategy != NULL)
  			FreeAccessStrategy(scan->rs_strategy);
  		scan->rs_strategy = NULL;
  	}
  	
  	/*
  		当扫描的表不是临时表并且扫描关系读取数据的大小超过Shared buffer的1/4时，就算是批量读了	
  	*/
  ```

- BAS_BULKWRITE

  ​	批量写的情况下，会分配16MB的空间(并且不超过Shared buffer的1/8)，原因是较小的ring buffer会频繁的进行wal flush，降低写入的效率。 

  ​	批量写的场景如下:

  - COPY FROM
  - CREATE TABLE AS
  - CREATE  MATERIALIZE VIEW或REFERSH MATERIALIZE VIEW
  - ALTER TABLE

  ```c
  /*
  	src/backend/access/heap/heapam.c
  */	
  /*
   * GetBulkInsertState - prepare status object for a bulk insert
   */
  
  BulkInsertState
  GetBulkInsertState(void)
  {
  	BulkInsertState bistate;
  
  	bistate = (BulkInsertState) palloc(sizeof(BulkInsertStateData));
  	bistate->strategy = GetAccessStrategy(BAS_BULKWRITE);
  	bistate->current_buf = InvalidBuffer;
  	return bistate;
  }
  
  /*
  	copy from、alter table、create table as等操作都属于Bulk Insert.
  */
  ```

- BAS_VACUUM

  Vacuum会分配256KB大小Ring buffer，如果有脏页需要wal flush然后写脏页，最后才能重用buffer。

  ```c
  /*
  	清理过程会用到Ring buffer
  */
  
  vacuum(List *relations, VacuumParams *params,BufferAccessStrategy bstrategy, bool isTopLevel)
  do_autovacuum(void)
  ```


### 三、Ring buffer场景测试

- BAS_BULKREAD

  ```sql
  -- 创建测试表
  postgres=# create sequence rb_seq increment by 1;
  CREATE SEQUENCE
  postgres=# create table rb_test(id int default nextval('rb_seq'),name text,dt timestamp);
  CREATE TABLE
  
  -- 加载数据
  postgres=# insert into rb_test(name,dt) select random_string(20),now() from generate_series(1,1000000);
  INSERT 0 1000000
  
  postgres=# vacuum rb_test;
  VACUUM
  
  test=# \d+
                           List of relations
   Schema |  Name   |   Type   |  Owner   |    Size    | Description 
  --------+---------+----------+----------+------------+-------------
   public | rb_seq  | sequence | postgres | 8192 bytes | 
   public | rb_test | table    | postgres | 50 MB      | 
   public | sd      | table    | postgres | 16 kB      | 
   public | st      | table    | postgres | 4360 kB    | 
   public | t       | table    | postgres | 8192 bytes | 
   public | test    | table    | postgres | 8192 bytes | 
  (6 rows)
  
  postgres=# show shared_buffers;
   shared_buffers 
  ----------------
   128MB
  (1 row)
  
  
  --清空shared_buffer
  [postgres@cituscn ~]$ pg_ctl restart
  
  -- 关闭并行，这里需要关闭并行否则pg_buffercache中缓存的元组就是Ring buffer的倍数
  postgres=# set max_parallel_workers_per_gather=0;
  SET
  
  -- 测试
  postgres=# select count(*) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
       0
  (1 row)
  
  postgres=# explain (analyze,verbose,costs,buffers,timing) select count(*) from rb_test;
                                                            QUERY PLAN                                                           
  -------------------------------------------------------------------------------------------------------------------------------
   Aggregate  (cost=20834.00..20834.01 rows=1 width=8) (actual time=119.225..119.226 rows=1 loops=1)
     Output: count(*)
     Buffers: shared read=8334
     ->  Seq Scan on public.rb_test  (cost=0.00..18334.00 rows=1000000 width=0) (actual time=0.036..75.585 rows=1000000 loops=1)
           Output: id, name, dt
           Buffers: shared read=8334
   Planning Time: 0.081 ms
   Execution Time: 119.278 ms
  (8 rows)
  
  postgres=# select count(*) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
      32
  (1 row)
  
  -- 正好等于256KB
  ```

- BAS_BULKWRITE

  ```sql
  -- 测试
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
       0
  (1 row)
  
  postgres=# set max_parallel_workers_per_gather=0;
  SET
  
  postgres=# create table t as select * from rb_test;
  SELECT 1000000
  postgres=# \d+
                             List of relations
   Schema |      Name      |   Type   | Owner  |    Size    | Description 
  --------+----------------+----------+--------+------------+-------------
   public | pg_buffercache | view     | flying | 0 bytes    | 
   public | rb_seq         | sequence | flying | 8192 bytes | 
   public | rb_test        | table    | flying | 50 MB      | 
   public | seq            | sequence | flying | 8192 bytes | 
   public | t              | table    | flying | 50 MB      | 
   public | test           | table    | flying | 56 kB      | 
  (6 rows)
  
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
      32
  (1 row)
  
  postgres=# select relfilenode,relforknumber,relblocknumber,isdirty,usagecount,pinning_backends from pg_buffercache where relfilenode='rb_test'::regclass and relforknumber!='0';
   relfilenode | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
  -------------+---------------+----------------+---------+------------+------------------
  (0 rows)
  
  
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='t';
   count 
  -------
    2080
  (1 row)
  
  postgres=# select relfilenode,relforknumber,relblocknumber,isdirty,usagecount,pinning_backends from pg_buffercache where relfilenode='t'::regclass and relforknumber!='0';
   relfilenode | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
  -------------+---------------+----------------+---------+------------+------------------
  (0 rows)
  
  
  
  -- 为什么会出现这种情况呢？(正常情况下应该是2048)
  postgres=# select a.relfilenode,a.relforknumber,a.relblocknumber  from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test' order by relblocknumber;
   relfilenode | relforknumber | relblocknumber 
  -------------+---------------+----------------
         49208 |             0 |           6338
         49208 |             0 |           6339
         49208 |             0 |           6340
         49208 |             0 |           6341
         49208 |             0 |           6342
         49208 |             0 |           6343
         49208 |             0 |           6344
         49208 |             0 |           6345
         49208 |             0 |           6346
         49208 |             0 |           6347
         49208 |             0 |           6348
         49208 |             0 |           6349
         49208 |             0 |           6350
         49208 |             0 |           6351
         49208 |             0 |           6352
         49208 |             0 |           6353
         49208 |             0 |           6354
         49208 |             0 |           6355
         49208 |             0 |           6356
         49208 |             0 |           6357
         49208 |             0 |           6358
         49208 |             0 |           6359
         49208 |             0 |           6360
         49208 |             0 |           6361
         49208 |             0 |           6362
         49208 |             0 |           6363
         49208 |             0 |           6364
         49208 |             0 |           6365
         49208 |             0 |           6366
         49208 |             0 |           6367
         49208 |             0 |           6368
         49208 |             0 |           6369
  (32 rows)
  
  
  postgres=# select a.relfilenode,a.relforknumber,a.relblocknumber  from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='t' and a.relblocknumber>=6338 and relblocknumber<=6369 order by relblocknumber ;
   relfilenode | relforknumber | relblocknumber 
  -------------+---------------+----------------
         57536 |             0 |           6338
         57536 |             0 |           6339
         57536 |             0 |           6340
         57536 |             0 |           6341
         57536 |             0 |           6342
         57536 |             0 |           6343
         57536 |             0 |           6344
         57536 |             0 |           6345
         57536 |             0 |           6346
         57536 |             0 |           6347
         57536 |             0 |           6348
         57536 |             0 |           6349
         57536 |             0 |           6350
         57536 |             0 |           6351
         57536 |             0 |           6352
         57536 |             0 |           6353
         57536 |             0 |           6354
         57536 |             0 |           6355
         57536 |             0 |           6356
         57536 |             0 |           6357
         57536 |             0 |           6358
         57536 |             0 |           6359
         57536 |             0 |           6360
         57536 |             0 |           6361
         57536 |             0 |           6362
         57536 |             0 |           6363
         57536 |             0 |           6364
         57536 |             0 |           6365
         57536 |             0 |           6366
         57536 |             0 |           6367
         57536 |             0 |           6368
         57536 |             0 |           6369
  (32 rows)
  
  -- 从上面查询可以看出，rb_test和b的某些页面有公共页面。
  -- 除去公共页面(rb_test所分配的256kb的buffer,所以批量写分配的Ring Buffer是16M)
  postgres=# select * from pg_buffercache where relblocknumber='6338';
   bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
  ----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
        197 |       49208 |          1663 |       13593 |             0 |           6338 | f       |          1 |                0
        419 |       57536 |          1663 |       13593 |             0 |           6338 | t       |          1 |                0
  (2 rows)
  -- 对于rb_test相当于批量读，所以缓存了32KB；对于t是批量写，所以缓存了16M
  
  -- 对于批量的update或者delete,会采用什么策略呢？
  
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
       0
  (1 row)
  postgres=# update rb_test set name=random_string(25);
  UPDATE 1000000
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
   16150
  (1 row)
  
  -- 可以看到没有采用Ring buffer策略
  -- README中提到批量的UPDATE或者DELETE会采取正常的策略
  In a scan that modifies every page in the scan, like a
  bulk UPDATE or DELETE, the buffers in the ring will always be dirtied and
  the ring strategy effectively degrades to the normal strategy.
  ```

- BAS_VACUUM

  ```sql
  -- 测试
  postgres=# set max_parallel_workers_per_gather=0;
  SET
  postgres=#  select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='tt';
   count 
  -------
       0
  (1 row)
  
  -- 如果vacuum一张数据没有变化的表，看不出当前的Buffer缓冲的策略(只扫描部分数据页和fsm、vm)
  postgres=#  vacuum tt;
  VACUUM
  
  postgres=#  select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='tt';
   count 
  -------
      37
  (1 row)
  
  postgres=# select relfilenode,relforknumber,relblocknumber,isdirty,usagecount,pinning_backends from pg_buffercache where relfilenode='tt'::regclass and relforknumber!='0';
   relfilenode | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
  -------------+---------------+----------------+---------+------------+------------------
         49227 |             2 |              0 | t       |          2 |                0
         49227 |             1 |              2 | f       |          5 |                0
         49227 |             1 |              3 | t       |          5 |                0
         49227 |             1 |              0 | t       |          1 |                0
         49227 |             1 |              1 | t       |          1 |                0
  (5 rows)
  
  
  /*
  	src/common/relpath.c
  	const char *const forkNames[] = {
  		"main",						/* MAIN_FORKNUM */
  		"fsm",						/* FSM_FORKNUM */
  		"vm",						/* VISIBILITYMAP_FORKNUM */
  		"init"						/* INIT_FORKNUM */
  	};
  
  */
  -- 可以看到fsm和vm加起来有5页，正好有32个块
  ```

  

