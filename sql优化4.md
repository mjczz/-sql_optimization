---
title: "sql优化4"
date: 2019-06-01
draft: true
---

### 业务场景 
class_out列表

### 原sql
```sql
SELECT s.student_name,s.student_code,s.parent_mobile,s.school,s.grade AS student_grade,co.*,na.realname,na.is_part,nl.lesson_id,
	                   nl.lesson_tag,nl.lesson_subject,nl.lesson_grade,nl.lesson_mode,nl.lesson_mode2,nl.lesson_type,
	                   nl.standard_group,nl.special_course,nl.unit_time,nl.lesson_name,nl.lesson_price
FROM nice_student_class_out AS co
LEFT JOIN nice_lesson AS nl ON co.lesson_id=nl.lesson_id
LEFT JOIN nice_student AS s ON co.student_id=s.student_id
LEFT JOIN nice_admin AS na ON na.userid=co.class_teacher_id
WHERE co.class_out_id > 0 AND co.class_status < '4' AND co.city ='1' AND co.class_campus ='108' AND co.ymd != 20171212 
```

### 查询计划分析

```sql
Limit  (cost=79694.65..79694.68 rows=10 width=578) (actual time=413.670..413.676 rows=10 loops=1)
  ->  Sort  (cost=79694.65..79697.25 rows=1037 width=578) (actual time=413.668..413.672 rows=10 loops=1)
        Sort Key: co.class_begin, co.class_out_id
        Sort Method: top-N heapsort  Memory: 30kB
        ->  Hash Left Join  (cost=3160.03..79672.24 rows=1037 width=578) (actual time=44.820..412.185 rows=923 loops=1)
              Hash Cond: (co.class_teacher_id = na.userid)
              ->  Hash Left Join  (cost=2796.44..79294.40 rows=1037 width=567) (actual time=41.756..408.527 rows=923 loops=1)
                    Hash Cond: (co.student_id = s.student_id)
                    ->  Hash Left Join  (cost=1046.38..77530.08 rows=1037 width=525) (actual time=19.509..385.472 rows=923 loops=1)
                          Hash Cond: (co.lesson_id = nl.lesson_id)
                          ->  Seq Scan on nice_student_class_out co  (cost=0.00..76469.44 rows=1037 width=433) (actual time=0.741..365.736 rows=923 loops=1)
                                Filter: ((class_out_id > 0) AND (class_status < '4'::smallint) AND (ymd <> 20171212) AND (city = '1'::smallint) AND (class_campus = '108'::smallint))
                                Rows Removed by Filter: 935973
                          ->  Hash  (cost=782.28..782.28 rows=21128 width=92) (actual time=18.589..18.589 rows=21128 loops=1)
                                Buckets: 32768  Batches: 1  Memory Usage: 2867kB
                                ->  Seq Scan on nice_lesson nl  (cost=0.00..782.28 rows=21128 width=92) (actual time=0.007..9.764 rows=21128 loops=1)
                    ->  Hash  (cost=1377.25..1377.25 rows=29825 width=46) (actual time=22.076..22.076 rows=29825 loops=1)
                          Buckets: 32768  Batches: 1  Memory Usage: 2654kB
                          ->  Seq Scan on nice_student s  (cost=0.00..1377.25 rows=29825 width=46) (actual time=0.007..11.085 rows=29825 loops=1)
              ->  Hash  (cost=313.26..313.26 rows=4026 width=15) (actual time=3.031..3.031 rows=4026 loops=1)
                    Buckets: 4096  Batches: 1  Memory Usage: 223kB
                    ->  Seq Scan on nice_admin na  (cost=0.00..313.26 rows=4026 width=15) (actual time=0.009..2.059 rows=4026 loops=1)

Planning time: 0.815 ms
Execution time: 413.941 ms
```

### 根据查询计划分析出
1\. 搜索条件顺序问题

### 优化后的sql
```sql
SELECT s.student_name,s.student_code,s.parent_mobile,s.school,s.grade AS student_grade,co.*,na.realname,na.is_part,nl.lesson_id,
	                   nl.lesson_tag,nl.lesson_subject,nl.lesson_grade,nl.lesson_mode,nl.lesson_mode2,nl.lesson_type,
	                   nl.standard_group,nl.special_course,nl.unit_time,nl.lesson_name,nl.lesson_price
FROM nice_student_class_out AS co
LEFT JOIN nice_lesson AS nl ON co.lesson_id=nl.lesson_id
LEFT JOIN nice_student AS s ON co.student_id=s.student_id
LEFT JOIN nice_admin AS na ON na.userid=co.class_teacher_id
WHERE   co.city ='1' AND co.class_campus ='108' AND co.class_status < '4' AND  co.ymd != 20171212 
```

### 优化后的sql查询计划分析

```sql
Hash Left Join  (cost=3160.03..77333.08 rows=1037 width=578) (actual time=56.142..375.398 rows=923 loops=1)
  Hash Cond: (co.class_teacher_id = na.userid)
  ->  Hash Left Join  (cost=2796.44..76955.24 rows=1037 width=567) (actual time=51.096..369.664 rows=923 loops=1)
        Hash Cond: (co.student_id = s.student_id)
        ->  Hash Left Join  (cost=1046.38..75190.92 rows=1037 width=525) (actual time=21.607..339.337 rows=923 loops=1)
              Hash Cond: (co.lesson_id = nl.lesson_id)
              ->  Seq Scan on nice_student_class_out co  (cost=0.00..74130.28 rows=1037 width=433) (actual time=0.600..317.289 rows=923 loops=1)
                    Filter: ((class_status < '4'::smallint) AND (ymd <> 20171212) AND (city = '1'::smallint) AND (class_campus = '108'::smallint))
                    Rows Removed by Filter: 935973
              ->  Hash  (cost=782.28..782.28 rows=21128 width=92) (actual time=20.819..20.819 rows=21128 loops=1)
                    Buckets: 32768  Batches: 1  Memory Usage: 2867kB
                    ->  Seq Scan on nice_lesson nl  (cost=0.00..782.28 rows=21128 width=92) (actual time=0.009..10.822 rows=21128 loops=1)
        ->  Hash  (cost=1377.25..1377.25 rows=29825 width=46) (actual time=29.308..29.308 rows=29825 loops=1)
              Buckets: 32768  Batches: 1  Memory Usage: 2654kB
              ->  Seq Scan on nice_student s  (cost=0.00..1377.25 rows=29825 width=46) (actual time=0.009..14.955 rows=29825 loops=1)
  ->  Hash  (cost=313.26..313.26 rows=4026 width=15) (actual time=5.000..5.000 rows=4026 loops=1)
        Buckets: 4096  Batches: 1  Memory Usage: 223kB
        ->  Seq Scan on nice_admin na  (cost=0.00..313.26 rows=4026 width=15) (actual time=0.014..3.452 rows=4026 loops=1)
Planning time: 0.709 ms
Execution time: 375.674 ms
```
### 结论，效率提升50ms左右
