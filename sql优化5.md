### 业务场景
资源在校区首次来源

### 原sql
```sql
SELECT ncl.source_from FROM	nice_crm2_log AS ncl
WHERE ncl.source_from > 0 AND ncl.student_id = '288599' AND ncl.campus = '202'
ORDER BY ncl. ID DESC
LIMIT 1 OFFSET 0
```

### 查询计划分析

```sql
Limit  (cost=30137.63..30137.64 rows=1 width=6) (actual time=237.177..237.178 rows=1 loops=1)
  ->  Sort  (cost=30137.63..30137.64 rows=1 width=6) (actual time=237.175..237.175 rows=1 loops=1)
        Sort Key: id DESC
        Sort Method: quicksort  Memory: 25kB
        ->  Seq Scan on nice_crm2_log ncl  (cost=0.00..30137.62 rows=1 width=6) (actual time=218.135..237.161 rows=1 loops=1)
              Filter: ((source_from > 0) AND (student_id = 288599) AND (campus = '202'::smallint))
              Rows Removed by Filter: 1166606
Planning time: 0.165 ms
Execution time: 237.204 ms
```

### 根据查询计划分析出
1\. 进行了全表扫描，没有键索引,要在student_id上建索引
2\. where条件顺序需要优化

```sql
-- 在student_id上建立索引
CREATE INDEX  CONCURRENTLY  "student_id_idx" ON "public"."nice_crm2_log" USING btree ("student_id");
```

### 优化后的sql
```sql
SELECT ncl.source_from FROM	nice_crm2_log AS ncl
WHERE ncl.student_id = '288599' AND ncl.campus = '202' and ncl.source_from > 0
ORDER BY ncl. ID DESC
LIMIT 1 OFFSET 0
```

### 优化后的sql查询计划分析

```sql
Limit  (cost=12.05..12.05 rows=1 width=6) (actual time=0.031..0.031 rows=1 loops=1)
  ->  Sort  (cost=12.05..12.05 rows=1 width=6) (actual time=0.030..0.030 rows=1 loops=1)
        Sort Key: id DESC
        Sort Method: quicksort  Memory: 25kB
        ->  Index Scan using student_id_idx on nice_crm2_log ncl  (cost=0.43..12.04 rows=1 width=6) (actual time=0.021..0.021 rows=1 loops=1)
              Index Cond: (student_id = 288599)
              Filter: ((source_from > 0) AND (campus = '202'::smallint))
Planning time: 0.112 ms
Execution time: 0.058 ms
```
### 结论，效率提升4000多倍

