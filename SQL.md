## 一、SQL语句分类

SQL（Structured Query Language，结构化查询语言）主要分为以下几类：

1. **DDL（数据定义语言）**

   * 用来定义数据库对象（库、表、视图、索引等）。
   * 常见语句：`CREATE`、`ALTER`、`DROP`。

2. **DML（数据操作语言）**

   * 用来操作表中的数据。
   * 常见语句：`INSERT`、`UPDATE`、`DELETE`。

3. **DQL（数据查询语言）**

   * 用来查询数据库中的数据。
   * 核心语句：`SELECT`。

4. **DCL（数据控制语言）**

   * 用来管理权限和安全。
   * 常见语句：`GRANT`、`REVOKE`。

5. **TCL（事务控制语言）**

   * 用来控制事务。
   * 常见语句：`COMMIT`、`ROLLBACK`、`SAVEPOINT`。

---

## 二、常用SQL查询模板

### 1. 基本查询

```sql
-- 查询所有字段
SELECT * FROM users;

-- 查询指定字段
SELECT id, name, age FROM users;
```

### 2. 条件查询

```sql
-- 查询年龄大于18的用户
SELECT * FROM users WHERE age > 18;

-- 查询年龄在18到30之间的用户
SELECT * FROM users WHERE age BETWEEN 18 AND 30;

-- 查询姓名以'A'开头的用户
SELECT * FROM users WHERE name LIKE 'A%';

-- 查询性别为男 且 年龄小于25
SELECT * FROM users WHERE gender = 'M' AND age < 25;
```

### 3. 排序与限制

```sql
-- 按年龄升序排序
SELECT * FROM users ORDER BY age ASC;

-- 按年龄降序，取前10条
SELECT * FROM users ORDER BY age DESC LIMIT 10;
```

### 4. 聚合查询

```sql
-- 统计总人数
SELECT COUNT(*) FROM users;

-- 计算平均年龄
SELECT AVG(age) FROM users;

-- 最大年龄和最小年龄
SELECT MAX(age), MIN(age) FROM users;
```

### 5. 分组查询

```sql
-- 按性别分组统计人数
SELECT gender, COUNT(*) FROM users GROUP BY gender;

-- 按性别分组，筛选人数大于5的组
SELECT gender, COUNT(*) 
FROM users 
GROUP BY gender 
HAVING COUNT(*) > 5;
```

### 6. 多表查询（JOIN）

```sql
-- 内连接：查询用户及其订单
SELECT u.id, u.name, o.order_id, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 左连接：查出所有用户及订单（没订单的用户显示NULL）
SELECT u.id, u.name, o.order_id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

### 7. 子查询

```sql
-- 查询年龄大于平均年龄的用户
SELECT * FROM users 
WHERE age > (SELECT AVG(age) FROM users);

-- 查询有订单的用户
SELECT * FROM users 
WHERE id IN (SELECT user_id FROM orders);
```

### 8. 修改与删除

```sql
-- 插入数据
INSERT INTO users (name, age, gender) VALUES ('Tom', 20, 'M');

-- 更新数据
UPDATE users SET age = 21 WHERE name = 'Tom';

-- 删除数据
DELETE FROM users WHERE age < 18;
```

---
好的👌 我帮你整理一个 **常见业务场景 SQL 模板清单**，再加上 **窗口函数（Window Function）** 的用法。这样你既有“日常常见SQL”也有“进阶面试必考点”。

---

## 三、常见业务场景 SQL 模板

### 1. 分页查询

```sql
-- 第2页，每页10条
SELECT * 
FROM users
ORDER BY id
LIMIT 10 OFFSET 10;
```

### 2. Top N 查询

```sql
-- 查询订单金额最高的前5个用户
SELECT user_id, SUM(amount) AS total_amount
FROM orders
GROUP BY user_id
ORDER BY total_amount DESC
LIMIT 5;
```

### 3. 查找重复数据

```sql
-- 查询姓名重复的用户
SELECT name, COUNT(*) 
FROM users
GROUP BY name
HAVING COUNT(*) > 1;
```

### 4. 查询最近7天活跃用户

```sql
SELECT DISTINCT user_id
FROM logins
WHERE login_time >= NOW() - INTERVAL '7 DAY';
```

### 5. 每天注册用户数

```sql
SELECT DATE(created_at) AS day, COUNT(*) AS new_users
FROM users
GROUP BY DATE(created_at)
ORDER BY day;
```

### 6. 某时间段内的订单总金额

```sql
SELECT SUM(amount) AS total_amount
FROM orders
WHERE created_at BETWEEN '2025-09-01' AND '2025-09-10';
```

---

## 四、窗口函数（Window Function）常用模板

窗口函数的语法：

```sql
函数名() OVER (PARTITION BY ... ORDER BY ...)
```

它不会把结果合并成一行，而是**在原始结果集的基础上增加计算列**。

---

### 1. 行号/排名

```sql
-- 给每个用户的订单按金额排序
SELECT user_id, order_id, amount,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS row_num,
       RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rank,
       DENSE_RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS dense_rank
FROM orders;
```

**区别**：

* `ROW_NUMBER`：严格递增（1,2,3…）。
* `RANK`：有并列会跳号（1,2,2,4…）。
* `DENSE_RANK`：有并列但不跳号（1,2,2,3…）。

---

### 2. 累计和/移动平均

```sql
-- 按日期累计订单金额
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- 最近3天的移动平均值
SELECT order_date, amount,
       AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM orders;
```

---

### 3. 分组内最大/最小值

```sql
-- 查询每个用户的最大订单金额
SELECT user_id, order_id, amount,
       MAX(amount) OVER (PARTITION BY user_id) AS max_amount
FROM orders;
```

---

### 4. 分组Top 1记录（搭配窗口函数和子查询）

```sql
-- 每个用户的最新订单
SELECT * FROM (
    SELECT user_id, order_id, created_at,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) t
WHERE rn = 1;
```

---

### 5. Lag/Lead（取前后行数据）

```sql
-- 计算订单金额与上一单的差值
SELECT user_id, order_id, amount,
       LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount,
       amount - LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS diff_with_prev
FROM orders;
```

---

## 五、子查询（Subquery）

**概念**：
子查询就是嵌套在 SQL 语句里的查询，可以出现在 `SELECT`、`FROM`、`WHERE` 等位置。

### 分类

1. **标量子查询**（返回单个值）

```sql
-- 查询工资高于平均工资的员工
SELECT * 
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

2. **IN 子查询**（返回多行）

```sql
-- 查询有订单的用户
SELECT * 
FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);
```

3. **EXISTS 子查询**（判断是否存在）

```sql
-- 查询至少有一个订单的用户
SELECT * 
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

4. **FROM 子查询（内联视图）**

```sql
-- 查询每个部门工资最高的员工
SELECT dept, MAX(salary) AS max_salary
FROM (
    SELECT dept, salary FROM employees
) t
GROUP BY dept;
```

👉 **适用场景**：

* 条件过滤（`IN`、`EXISTS`）。
* 计算中间结果（`FROM` 子查询）。
* 返回标量结果（`SELECT` 子查询）。

---

## 六、CTE（Common Table Expression，公用表表达式）

**概念**：
CTE 用 `WITH` 语句定义一个临时结果集，类似“命名子查询”，可以在主查询中多次引用，提升可读性。

语法：

```sql
WITH cte_name AS (
    子查询SQL
)
SELECT * FROM cte_name;
```

### 示例

1. **简化复杂查询**

```sql
WITH dept_salary AS (
    SELECT dept, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept
)
SELECT e.name, e.salary, d.avg_salary
FROM employees e
JOIN dept_salary d ON e.dept = d.dept
WHERE e.salary > d.avg_salary;
```

2. **递归 CTE**（常用于层级结构、树形查询）

```sql
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id
    FROM employees
    WHERE id = 1   -- 起点：某个员工

    UNION ALL

    SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

👉 **适用场景**：

* 把复杂 SQL 拆成多个步骤，便于理解。
* 实现递归查询（组织架构、分类树）。

---


