### 业务场景
查询学生在原教师产生的课耗

### 原sql
```sql
SELECT co.class_teacher_id,co.student_id,sum(co.keshi_num) sum_keshi_num
FROM nice_student_class_out as co
WHERE co.class_status = 4 AND co.class_teacher_id > 0
GROUP BY co.class_teacher_id,co.student_id
```

### 分析
这条sql属于业务层没有加条件限制
```php
// 变更老师--学生在原教师产生的课耗
    public function old_teacher_kehao($param)
    {
        $wherestr = " WHERE co.class_status = 4 AND co.class_teacher_id > 0";

        if(!empty($param['in_class_teacher_id'])){
            $wherestr .= " AND co.class_teacher_id IN (" . implode(',',$param['in_class_teacher_id']) . ")";
        }

        $sql = "SELECT co.class_teacher_id,co.student_id,sum(co.keshi_num) sum_keshi_num
                FROM nice_student_class_out as co 
                {$wherestr}
                GROUP BY co.class_teacher_id,co.student_id";

        $res = $this->db->query($sql)->result_array();
        return $res;
    }
```

### 更改业务层代码
```php 
 // 变更老师--学生在原教师产生的课耗
    public function old_teacher_kehao($param)
    {
        if (empty($param['in_class_teacher_id'])) {return 0;}
        $wherestr = " WHERE co.class_status = 4 AND co.class_teacher_id > 0";

        $wherestr .= " AND co.class_teacher_id IN (" . implode(',',$param['in_class_teacher_id']) . ")";

        $sql = "SELECT co.class_teacher_id,co.student_id,sum(co.keshi_num) sum_keshi_num
                FROM nice_student_class_out as co 
                {$wherestr}
                GROUP BY co.class_teacher_id,co.student_id";

        $res = $this->db->query($sql)->result_array();
        return $res;
    }
```
