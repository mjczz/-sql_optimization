### 业务场景
资源列表，不传参数的情况下(极端情况，一般都会有参数)

### 原sql
```sql
SELECT	ncs.*, nst.status AS student_status,
	ncc.assistant AS campus_assistant,
	ncc.follow_time AS campus_follow_time,
	ncc.next_time AS campus_next_time,
	ncc.campus_cuid,
	ncc.campus_ctime,
	ncc.follow_num AS campus_follow_num,
	ncc.distribution_time,
	ncc.campus AS current_campus,
	ncc.campus_source_from,
	ncc.campus_source_child_from,
	ncc. ID crm_campus_id,
	ncc.campus_status,
	ncc.campus_is_sc,
	ncc.campus_is_yx
FROM nice_crm2_campus AS ncc
LEFT JOIN nice_crm2_student AS ncs ON ncs.student_id = ncc.student_id
LEFT JOIN nice_student AS nst ON ncs.s_id = nst.student_id
ORDER BY ncs.student_id DESC
LIMIT 50 OFFSET 0
```

### 查询计划分析

```sql
Limit  (cost=171534.88..171535.01 rows=50 width=267) (actual time=4326.333..4326.356 rows=50 loops=1)
  ->  Sort  (cost=171534.88..173003.35 rows=587388 width=267) (actual time=4326.331..4326.346 rows=50 loops=1)
        Sort Key: ncs.student_id DESC
        Sort Method: top-N heapsort  Memory: 39kB
        ->  Hash Left Join  (cost=83648.23..152022.28 rows=587388 width=267) (actual time=940.417..3725.608 rows=587392 loops=1)
              Hash Cond: (ncs.s_id = nst.student_id)
              ->  Hash Left Join  (cost=81898.17..142195.63 rows=587388 width=265) (actual time=922.628..3444.172 rows=587392 loops=1)
                    Hash Cond: (ncc.student_id = ncs.student_id)
                    ->  Seq Scan on nice_crm2_campus ncc  (cost=0.00..13125.88 rows=587388 width=48) (actual time=0.041..179.576 rows=587392 loops=1)
                    ->  Hash  (cost=41250.74..41250.74 rows=950274 width=221) (actual time=789.739..789.739 rows=950218 loops=1)
                          Buckets: 16384 (originally 16384)  Batches: 128 (originally 64)  Memory Usage: 4045kB
                          ->  Seq Scan on nice_crm2_student ncs  (cost=0.00..41250.74 rows=950274 width=221) (actual time=0.017..251.939 rows=950218 loops=1)
              ->  Hash  (cost=1377.25..1377.25 rows=29825 width=6) (actual time=17.640..17.640 rows=29825 loops=1)
                    Buckets: 32768  Batches: 1  Memory Usage: 1422kB
                    ->  Seq Scan on nice_student nst  (cost=0.00..1377.25 rows=29825 width=6) (actual time=0.009..11.521 rows=29825 loops=1)
Planning time: 0.565 ms
Execution time: 4326.667 ms
```

### 根据查询计划分析出
1\. 全表扫描了nice_student_nst，29825行数据
2\. 全表扫描了nice_crm2_campus,587392条数据
3\. 排序字段用了nice_crm2_studnet的student_id

### 优化后的sql
将排序字段改为ncc.student_id

```sql
SELECT	ncs.*, nst.status AS student_status,
	ncc.assistant AS campus_assistant,
	ncc.follow_time AS campus_follow_time,
	ncc.next_time AS campus_next_time,
	ncc.campus_cuid,
	ncc.campus_ctime,
	ncc.follow_num AS campus_follow_num,
	ncc.distribution_time,
	ncc.campus AS current_campus,
	ncc.campus_source_from,
	ncc.campus_source_child_from,
	ncc. ID crm_campus_id,
	ncc.campus_status,
	ncc.campus_is_sc,
	ncc.campus_is_yx
FROM nice_crm2_campus AS ncc
LEFT JOIN nice_crm2_student AS ncs ON ncs.student_id = ncc.student_id
LEFT JOIN nice_student AS nst ON ncs.s_id = nst.student_id
ORDER BY ncs.student_id DESC
LIMIT 50 OFFSET 0
```

### 优化后的sql查询计划分析

```sql
Limit  (cost=1.14..55.93 rows=50 width=271) (actual time=0.020..0.294 rows=50 loops=1)
  ->  Nested Loop Left Join  (cost=1.14..643722.83 rows=587388 width=271) (actual time=0.020..0.292 rows=50 loops=1)
        ->  Nested Loop Left Join  (cost=0.85..455748.39 rows=587388 width=269) (actual time=0.016..0.195 rows=50 loops=1)
              ->  Index Scan Backward using crm2_campus_student_id_asc on nice_crm2_campus ncc  (cost=0.42..45947.32 rows=587388 width=48) (actual time=0.008..0.028 rows=50 loops=1)
              ->  Index Scan using nice_crm2_student_pkey on nice_crm2_student ncs  (cost=0.42..0.69 rows=1 width=221) (actual time=0.002..0.002 rows=1 loops=50)
                    Index Cond: (student_id = ncc.student_id)
        ->  Index Scan using nice_student_pkey on nice_student nst  (cost=0.29..0.31 rows=1 width=6) (actual time=0.001..0.001 rows=0 loops=50)
              Index Cond: (ncs.s_id = student_id)
Planning time: 0.479 ms
Execution time: 0.430 ms
```

### 结论
1\. 根据主表nice_crm2_campus的student_id排序，效率明细提升1万多倍
2\. 另外，测试了带查询条件的情况，效率也提升1倍多

