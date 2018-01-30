```
mysql> show variables like '%query%';
+------------------------------+----------------------------------+
| Variable_name                | Value                            |
+------------------------------+----------------------------------+
| binlog_rows_query_log_events | OFF                              |
| ft_query_expansion_limit     | 20                               |
| have_query_cache             | YES                              |
| long_query_time              | 10.000000                        |
| query_alloc_block_size       | 8192                             |
| query_cache_limit            | 1048576                          |
| query_cache_min_res_unit     | 4096                             |
| query_cache_size             | 1048576                          |
| query_cache_type             | OFF                              |
| query_cache_wlock_invalidate | OFF                              |
| query_prealloc_size          | 8192                             |
| slow_query_log               | OFF                              |
| slow_query_log_file          | /var/lib/mysql/node_102-slow.log |
+------------------------------+----------------------------------+
13 rows in set (0.00 sec)

mysql>
```

query\_cache\_type: ON, OFF, DEMAND（要在sql语句中明确指定sql\_cache才会缓存）



* 查询相关的状态变量

```
mysql> SHOW GLOBAL STATUS LIKE 'Qcache%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Qcache_free_blocks      | 1       |
| Qcache_free_memory      | 1031832 |
| Qcache_hits             | 0       |
| Qcache_inserts          | 0       |
| Qcache_lowmem_prunes    | 0       |
| Qcache_not_cached       | 232     |
| Qcache_queries_in_cache | 0       |
| Qcache_total_blocks     | 1       |
+-------------------------+---------+
8 rows in set (0.01 sec)

mysql> 

```



