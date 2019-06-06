---
title: "sql优化1"
date: 2019-06-01
draft: true
---

### 原sql
```sql
SELECT co.class_teacher_id,na.realname
FROM nice_student_class_out AS co
LEFT JOIN nice_student AS s ON s.student_id = co.student_id
LEFT JOIN nice_lesson AS nl ON nl.lesson_id = co.lesson_id
LEFT JOIN nice_admin AS na ON co.class_teacher_id=na.userid
WHERE co.class_out_id > 0 AND co.class_campus ='201' 
AND co.class_time BETWEEN '2019-06-01 00:00:00' AND '2019-06-01 23:59:00'  AND co.ymd != 20171212
group by co.class_teacher_id,na.realname
LIMIT 1000 OFFSET 0
```

### 查询计划分析

```sql
Limit  (cost=76786.35..76786.82 rows=62 width=13) (actual time=793.696..793.777 rows=45 loops=1)
  ->  Group  (cost=76786.35..76786.82 rows=62 width=13) (actual time=793.694..793.770 rows=45 loops=1)
        Group Key: co.class_teacher_id, na.realname
        ->  Sort  (cost=76786.35..76786.51 rows=62 width=13) (actual time=793.692..793.728 rows=212 loops=1)
              Sort Key: co.class_teacher_id, na.realname
              Sort Method: quicksort  Memory: 34kB
              ->  Nested Loop Left Join  (cost=0.28..76784.50 rows=62 width=13) (actual time=0.836..793.555 rows=212 loops=1)
                    ->  Seq Scan on nice_student_class_out co  (cost=0.00..76469.44 rows=62 width=12) (actual time=0.827..792.874 rows=212 loops=1)
                          Filter: ((class_out_id > 0) AND ((class_time)::text >= '2019-06-01 00:00:00'::text) AND ((class_time)::text <= '2019-06-01 23:59:00'::text) AND (ymd <> 20171212) AND (class_campus = '201'::smallint))
                          Rows Removed by Filter: 936684
                    ->  Index Scan using nice_admin_pkey on nice_admin na  (cost=0.28..5.07 rows=1 width=13) (actual time=0.002..0.002 rows=1 loops=212)
                          Index Cond: (co.class_teacher_id = userid)
Planning time: 0.505 ms
Execution time: 793.860 ms
```

### 根据查询计划分析出
1\. 多余的查询条件 class_out_id > 0 
2\. class_time筛选时间性能低

### 优化后的sql
```sql
SELECT co.class_teacher_id,na.realname
FROM nice_student_class_out AS co
LEFT JOIN nice_student AS s ON s.student_id = co.student_id
LEFT JOIN nice_lesson AS nl ON nl.lesson_id = co.lesson_id
LEFT JOIN nice_admin AS na ON co.class_teacher_id=na.userid
WHERE co.class_campus ='201'  
AND co.class_begin >= '1559318400' AND co.class_begin <= '1559404740'  
AND co.ymd != 20171212
group by co.class_teacher_id,na.realname
LIMIT 1000 OFFSET 0
```

### 优化后的sql查询计划分析

```sql
Limit  (cost=16157.16..16158.36 rows=160 width=13) (actual time=12.351..12.429 rows=45 loops=1)
  ->  Group  (cost=16157.16..16158.36 rows=160 width=13) (actual time=12.350..12.419 rows=45 loops=1)
        Group Key: co.class_teacher_id, na.realname
        ->  Sort  (cost=16157.16..16157.56 rows=160 width=13) (actual time=12.347..12.379 rows=212 loops=1)
              Sort Key: co.class_teacher_id, na.realname
              Sort Method: quicksort  Memory: 34kB
              ->  Hash Left Join  (cost=496.63..16151.31 rows=160 width=13) (actual time=4.964..12.262 rows=212 loops=1)
                    Hash Cond: (co.class_teacher_id = na.userid)
                    ->  Bitmap Heap Scan on nice_student_class_out co  (cost=133.04..15785.52 rows=160 width=12) (actual time=1.847..9.028 rows=212 loops=1)
                          Recheck Cond: ((class_begin >= 1559318400) AND (class_begin <= 1559404740))
                          Filter: ((ymd <> 20171212) AND (class_campus = '201'::smallint))
                          Rows Removed by Filter: 6793
                          Heap Blocks: exact=3527
                          ->  Bitmap Index Scan on class_begin_idx  (cost=0.00..133.00 rows=5258 width=0) (actual time=1.194..1.194 rows=7005 loops=1)
                                Index Cond: ((class_begin >= 1559318400) AND (class_begin <= 1559404740))
                    ->  Hash  (cost=313.26..313.26 rows=4026 width=13) (actual time=3.080..3.080 rows=4026 loops=1)
                          Buckets: 4096  Batches: 1  Memory Usage: 218kB
                          ->  Seq Scan on nice_admin na  (cost=0.00..313.26 rows=4026 width=13) (actual time=0.011..1.690 rows=4026 loops=1)
Planning time: 0.332 ms
Execution time: 12.506 ms
```
### 结论，效率提升70倍
