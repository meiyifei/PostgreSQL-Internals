# 		Ring buffer策略使用场景解读

原文地址:<https://yq.aliyun.com/articles/697694> 

### 一、Ring buffer介绍

```bash
#src/backend/storage/buffer/README

Buffer Ring Replacement Strategy
环形缓冲区更换策略(某些情况下将shared buffer替换为Ring Buffer)
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
```

​	在读写大表时，PostgreSQL将使用Ring buffer而不是Shared Buffer。ring buffer就是在当需要大量访问页面的情况下如vacuum或者大量的全表扫描时采用的一种特殊的策略。不会像正常的时钟扫描算法交换出整个缓冲区，而是在一小块缓冲区上使用时钟扫描算法，并会重用缓冲区完成整个扫描。这样就会避免大量全表扫描带来的缓冲区命中率的下降。

### 二、Ring buffer策略使用场景

什么时候会使用到Ring buffer策略呢？

```bash
#src/backend/storage/buffer/freelist.c

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

可以看到在某些场景下会使用Ring buffer策略:

```bash
#src/include/storage/bufmgr.h

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

  ```bash
  #src/backend/access/heap/heapam.c
  
  if (!RelationUsesLocalBuffers(scan->rs_rd) &&
  		scan->rs_nblocks > NBuffers / 4)
  	{
  		allow_strat = scan->rs_allow_strat;
  		allow_sync = scan->rs_allow_sync;
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
  	
  	-- 当扫描的表不是临时表并且扫描关系读取数据的大小超过Shared buffer的1/4时，就算是批量读了。
  ```

- BAS_BULKWRITE

  ​	批量写的情况下，会分配16MB的空间(并且不超过Shared buffer的1/8)，原因是较小的ring buffer会频繁的进行wal flush，降低写入的效率。 

  ​	批量写的场景如下:

  - COPY FROM
  - CREATE TABLE AS
  - CREATE  MATERIALIZE VIEW或REFERSH MATERIALIZE VIEW
  - ALTER TABLE

- BAS_VACUUM

  Vacuum会分配256KB大小Ring buffer，如果有脏页需要wal flush然后写脏页，最后才能重用buffer。

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
  
  postgres=# \d+
                          List of relations
   Schema |  Name   |   Type   | Owner  |    Size    | Description 
  --------+---------+----------+--------+------------+-------------
   public | rb_seq  | sequence | flying | 8192 bytes | 
   public | rb_test | table    | flying | 65 MB      | 
   public | seq     | sequence | flying | 8192 bytes | 
   public | test    | table    | flying | 56 kB      | 
  (4 rows)
  
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
  
  postgres=# create table t as select * from rb_test;
  SELECT 1000000
  postgres=# \d+
                             List of relations
   Schema |      Name      |   Type   | Owner  |    Size    | Description 
  --------+----------------+----------+--------+------------+-------------
   public | pg_buffercache | view     | flying | 0 bytes    | 
   public | rb_seq         | sequence | flying | 8192 bytes | 
   public | rb_test        | table    | flying | 65 MB      | 
   public | seq            | sequence | flying | 8192 bytes | 
   public | t              | table    | flying | 65 MB      | 
   public | test           | table    | flying | 56 kB      | 
  (6 rows)
  
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
   count 
  -------
      32
  (1 row)
  
  postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='t';
   count 
  -------
    2048
  (1 row)
  
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
postgres=# vacuum freeze rb_test;
VACUUM
postgres=# select count(1) from pg_buffercache as a,pg_class as b where a.relfilenode=b.relfilenode and b.relname='rb_test';
 count 
-------
    40
(1 row)

postgres=# select a.relfilenode,a.relforknumber,a.relblocknumber,a.isdirty,a.usagecount,a.pinning_backends from pg_buffercache as a,pg_class as b where a.lname='rb_test' and a.relforknumber!='0';
 relfilenode | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
-------------+---------------+----------------+---------+------------+------------------
      139882 |             2 |              0 | t       |          2 |                0
      139882 |             1 |              2 | f       |          5 |                0
      139882 |             1 |              3 | f       |          5 |                0
      139882 |             1 |              4 | f       |          5 |                0
      139882 |             1 |              6 | f       |          2 |                0
      139882 |             1 |              0 | f       |          1 |                0
      139882 |             1 |              1 | f       |          1 |                0
      139882 |             1 |              5 | f       |          1 |                0
(8 rows)

-- 可以看到fsm和vm加起来有8页，正好有32个块
```

