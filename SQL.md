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

## 三、常见业务场景 SQL查询模板

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

## 七、SQL 日期与时间函数详解（MySQL 为例）

SQL 的日期函数可以分为以下几类：

| 功能分类      | 常用函数                                   | 示例        |
| --------- | -------------------------------------- | --------- |
| 获取当前日期/时间 | `NOW()`、`CURDATE()`、`CURTIME()`        | 获取当前时间、日期 |
| 日期提取      | `YEAR()`、`MONTH()`、`DAY()`、`HOUR()`    | 从日期中提取部分  |
| 日期格式化     | `DATE_FORMAT()`                        | 按格式输出日期   |
| 日期运算      | `DATE_ADD()`、`DATE_SUB()`、`DATEDIFF()` | 时间加减      |
| 时间差计算     | `TIMESTAMPDIFF()`、`TIMEDIFF()`         | 两个时间差值    |
| 时间戳转换     | `UNIX_TIMESTAMP()`、`FROM_UNIXTIME()`   | 日期与时间戳互转  |

---

### 1. 获取当前日期和时间

| 函数          | 说明               | 示例                                      |
| ----------- | ---------------- | --------------------------------------- |
| `NOW()`     | 当前日期 + 时间（精确到秒）  | `SELECT NOW();` → `2025-10-12 09:32:10` |
| `CURDATE()` | 当前日期             | `SELECT CURDATE();` → `2025-10-12`      |
| `CURTIME()` | 当前时间             | `SELECT CURTIME();` → `09:32:10`        |
| `SYSDATE()` | 执行时刻的时间（多次执行会变化） | `SELECT SYSDATE();`                     |

🔸 区别说明：

* `NOW()`：在语句开始执行时固定。
* `SYSDATE()`：在函数执行瞬间取值，可能略有差异。

---

### 2. 提取日期的组成部分

| 函数                | 含义          | 示例                                | 结果     |
| ----------------- | ----------- | --------------------------------- | ------ |
| `YEAR(date)`      | 提取年份        | `SELECT YEAR('2025-10-12');`      | 2025   |
| `MONTH(date)`     | 提取月份        | `SELECT MONTH('2025-10-12');`     | 10     |
| `DAY(date)`       | 提取日         | `SELECT DAY('2025-10-12');`       | 12     |
| `HOUR(time)`      | 提取小时        | `SELECT HOUR('10:45:00');`        | 10     |
| `MINUTE(time)`    | 提取分钟        | `SELECT MINUTE('10:45:00');`      | 45     |
| `SECOND(time)`    | 提取秒         | `SELECT SECOND('10:45:00');`      | 0      |
| `DAYOFWEEK(date)` | 返回星期（1=星期日） | `SELECT DAYOFWEEK('2025-10-12');` | 1      |
| `DAYNAME(date)`   | 返回星期名称      | `SELECT DAYNAME('2025-10-12');`   | Sunday |
| `DAYOFYEAR(date)` | 返回一年中的第几天   | `SELECT DAYOFYEAR('2025-10-12');` | 285    |
| `WEEK(date)`      | 返回一年中的第几周   | `SELECT WEEK('2025-10-12');`      | 41     |

---

### 3. 日期格式化

 `DATE_FORMAT(date, format)`

将日期按指定格式输出（最常用）

```sql
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');
```

常用格式符：

| 格式符  | 含义        | 示例结果    |
| ---- | --------- | ------- |
| `%Y` | 年（4位）     | 2025    |
| `%y` | 年（2位）     | 25      |
| `%m` | 月（01–12）  | 10      |
| `%d` | 日（01–31）  | 12      |
| `%H` | 小时（00–23） | 09      |
| `%i` | 分钟（00–59） | 32      |
| `%s` | 秒（00–59）  | 10      |
| `%W` | 星期几       | Sunday  |
| `%M` | 月份英文名     | October |

---

### 4. 日期运算

 `DATE_ADD(date, INTERVAL expr unit)`

向日期增加时间间隔。

```sql
SELECT DATE_ADD('2025-10-12', INTERVAL 7 DAY);  -- 加7天
SELECT DATE_ADD('2025-10-12', INTERVAL 1 MONTH); -- 加1个月
```

 `DATE_SUB(date, INTERVAL expr unit)`

从日期中减去时间间隔。

```sql
SELECT DATE_SUB('2025-10-12', INTERVAL 10 DAY); -- 减10天
```

 可用时间单位

| 单位     | 含义 |
| ------ | -- |
| SECOND | 秒  |
| MINUTE | 分钟 |
| HOUR   | 小时 |
| DAY    | 天  |
| WEEK   | 周  |
| MONTH  | 月  |
| YEAR   | 年  |

---

### 5. 日期差值计算

 `DATEDIFF(date1, date2)`

计算两个日期相差的天数（date1 - date2）

```sql
SELECT DATEDIFF('2025-10-12', '2025-09-30');  -- 12
```

 `TIMESTAMPDIFF(unit, datetime1, datetime2)`

按指定单位计算时间差。

```sql
SELECT TIMESTAMPDIFF(DAY, '2025-09-30', '2025-10-12');  -- 12天
SELECT TIMESTAMPDIFF(HOUR, '2025-09-30 00:00:00', '2025-10-12 12:00:00');  -- 小时差
```

---

### 6. 时间戳与日期互转

 `UNIX_TIMESTAMP([datetime])`

将日期时间转换为 Unix 时间戳（秒）

```sql
SELECT UNIX_TIMESTAMP('2025-10-12 09:30:00');  -- 1750000000（示例）
```

 `FROM_UNIXTIME(timestamp)`

将时间戳转为日期格式

```sql
SELECT FROM_UNIXTIME(1750000000);  -- '2025-10-12 09:30:00'
```

 与格式化结合

```sql
SELECT FROM_UNIXTIME(UNIX_TIMESTAMP(), '%Y-%m-%d');
```

---

###  7. 综合应用示例

```sql
-- 查询过去7天内注册的用户
SELECT *
FROM users
WHERE register_time >= DATE_SUB(NOW(), INTERVAL 7 DAY);

-- 查询每月新增用户数量
SELECT DATE_FORMAT(register_time, '%Y-%m') AS month,
       COUNT(*) AS user_count
FROM users
GROUP BY month
ORDER BY month;

-- 计算两次登录之间的时间差（分钟）
SELECT user_id,
       TIMESTAMPDIFF(MINUTE, last_login, current_login) AS diff_minutes
FROM login_log;
```

---

###  8. 总结对比表

| 函数                 | 功能      | 示例                                    |
| ------------------ | ------- | ------------------------------------- |
| `NOW()`            | 当前时间    | `2025-10-12 09:32:10`                 |
| `CURDATE()`        | 当前日期    | `2025-10-12`                          |
| `DATE_ADD()`       | 日期加     | `DATE_ADD(NOW(), INTERVAL 1 DAY)`     |
| `DATE_SUB()`       | 日期减     | `DATE_SUB(NOW(), INTERVAL 1 MONTH)`   |
| `DATEDIFF()`       | 两日期相差天数 | `DATEDIFF('2025-10-12','2025-09-30')` |
| `TIMESTAMPDIFF()`  | 指定单位差   | `TIMESTAMPDIFF(HOUR, a, b)`           |
| `DATE_FORMAT()`    | 格式化日期   | `DATE_FORMAT(NOW(),'%Y/%m/%d')`       |
| `UNIX_TIMESTAMP()` | 日期→时间戳  | `UNIX_TIMESTAMP(NOW())`               |
| `FROM_UNIXTIME()`  | 时间戳→日期  | `FROM_UNIXTIME(1697073600)`           |

---

明白了，你问的是 **`WHERE` 条件里可以写什么类型的条件**，我给你系统梳理一下。

---

## 八、SQL 中 `WHERE` 的条件类型

`WHERE` 是用来 **过滤原始表的行**，支持多种条件类型，包括 **比较、逻辑、范围、模式、空值、子查询等**。

---

### 1.  比较运算符（Comparison Operators）

| 运算符     | 说明   | 示例                     |
| ------- | ---- | ---------------------- |
| =       | 等于   | `WHERE age = 18`       |
| <> 或 != | 不等于  | `WHERE age <> 18`      |
| >       | 大于   | `WHERE salary > 5000`  |
| >=      | 大于等于 | `WHERE salary >= 5000` |
| <       | 小于   | `WHERE score < 60`     |
| <=      | 小于等于 | `WHERE score <= 60`    |

---

### 2.  逻辑运算符（Logical Operators）

| 运算符 | 说明 | 示例                                        |
| --- | -- | ----------------------------------------- |
| AND | 并且 | `WHERE age > 18 AND gender='M'`           |
| OR  | 或者 | `WHERE age < 18 OR age > 60`              |
| NOT | 非  | `WHERE NOT active` 或 `WHERE NOT (age>18)` |

---

### 3. 范围运算符

| 运算符                 | 说明      | 示例                                |
| ------------------- | ------- | --------------------------------- |
| BETWEEN ... AND ... | 区间，包括边界 | `WHERE age BETWEEN 18 AND 30`     |
| IN (...)            | 值集合匹配   | `WHERE department IN ('HR','IT')` |

---

### 4. 模式匹配（Pattern Matching）

| 运算符      | 说明   | 示例                                                               |
| -------- | ---- | ---------------------------------------------------------------- |
| LIKE     | 模糊匹配 | `WHERE name LIKE 'A%'`（以A开头）<br>`WHERE name LIKE '%son'`（以son结尾） |
| NOT LIKE | 非匹配  | `WHERE name NOT LIKE 'A%'`                                       |

**可用通配符**：

* `%` 表示任意长度字符
* `_` 表示单个字符

---

### 5.  空值判断

| 运算符         | 说明   | 示例                         |
| ----------- | ---- | -------------------------- |
| IS NULL     | 判断为空 | `WHERE deleted_at IS NULL` |
| IS NOT NULL | 判断非空 | `WHERE email IS NOT NULL`  |

---

### 6. 子查询条件

`WHERE` 可以引用子查询结果：

```sql
-- 查询有订单的用户
SELECT * 
FROM users
WHERE id IN (SELECT user_id FROM orders);

-- 查询工资大于部门平均值的员工
SELECT * 
FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE dept = e.dept);
```

---

### 7. 其他条件（高级用法）

* **算术表达式**：

```sql
WHERE (salary * 12) > 60000
```

* **字符串函数**：

```sql
WHERE LENGTH(name) > 3
```

* **日期/时间函数**：

```sql
WHERE created_at >= '2025-01-01'
```

---

## 九、 `WHERE` 和 `HAVING` 的区别

| 特性      | WHERE                 | HAVING                |
| ------- | --------------------- | --------------------- |
| 作用对象    | **原始表的数据行**           | **分组后的聚合结果**          |
| 能否用聚合函数 | ❌ 不行（在 SQL 标准里）       | ✅ 可以                  |
| 执行顺序    | 在 **GROUP BY 前** 过滤数据 | 在 **GROUP BY 后** 过滤数据 |

**顺序理解：**

1. `FROM` → 选择表
2. `WHERE` → 对原始行过滤
3. `GROUP BY` → 分组
4. 聚合函数计算 → `COUNT`, `SUM`, `AVG` 等
5. `HAVING` → 对聚合结果再过滤
6. `SELECT` → 输出列
7. `ORDER BY` → 排序

---

### HAVING例子

你的 SQL：

```sql
SELECT num
FROM `number`
GROUP BY num
HAVING COUNT(num) = 1;
```

分析：

1. `GROUP BY num` 会把表按 `num` 分组。
2. `COUNT(num)` 会统计每个 `num` 出现的次数。
3. `HAVING COUNT(num) = 1` 会保留只出现一次的 `num`。

如果你写成 `WHERE COUNT(num) = 1` 就会报错，因为 `COUNT(num)` 是 **聚合函数**，它要在分组后才有值，`WHERE` 阶段无法访问。

---

### 举例对比

假设表 `number`：

| num |
| --- |
| 1   |
| 2   |
| 2   |
| 3   |

```sql
-- 1. 使用 WHERE （错误示例）
SELECT num
FROM number
WHERE COUNT(num) = 1  -- ❌ 聚合函数不能用在这里
GROUP BY num;

-- 2. 使用 HAVING （正确）
SELECT num
FROM number
GROUP BY num
HAVING COUNT(num) = 1;  -- ✅ 返回 1, 3
```

---

💡 **总结规律**：

* `WHERE` → 用于**逐行过滤原始数据**，聚合函数不能用。
* `HAVING` → 用于**分组后的过滤**，可以用聚合函数。
* 如果你只是想过滤某列的普通条件（比如 `num > 5`），用 `WHERE`；
* 如果你要过滤“每组的统计值”（比如出现次数、平均值），用 `HAVING`。

---
明白了，你问的是 **SQL 中算术表达式可以做哪些运算**，我给你整理得详细一点。

---

## 十、SQL 算术表达式支持的运算

SQL 的算术表达式可以在 `SELECT`、`WHERE`、`HAVING`、`ORDER BY` 等子句中使用，主要支持 **基本算术运算、取模、优先级控制**，以及部分数据库支持的 **函数运算**。

---

### 1. 基本算术运算符

| 运算符           | 说明 | 示例                                                                                        |
| ------------- | -- | ----------------------------------------------------------------------------------------- |
| `+`           | 加  | `SELECT salary + bonus AS total_income FROM employees;`                                   |
| `-`           | 减  | `SELECT salary - tax AS net_income FROM employees;`                                       |
| `*`           | 乘  | `SELECT price * quantity AS total_price FROM orders;`                                     |
| `/`           | 除  | `SELECT salary / 12 AS monthly_salary FROM employees;`                                    |
| `%` 或 `MOD()` | 取模 | `SELECT id % 2 AS remainder FROM users;` <br>`SELECT MOD(id, 2) AS remainder FROM users;` |

> 注意：不同数据库取模运算符可能不同：
>
> * MySQL/PostgreSQL：`%` 或 `MOD(a,b)`
> * SQL Server：`%`

---

### 2. 优先级和括号

SQL 的算术运算遵循标准优先级：

1. `()` 括号
2. `*`、`/`、`%`
3. `+`、`-`

示例：

```sql
SELECT (salary + bonus) * 12 AS annual_income FROM employees;
```

括号确保先加后乘。

---

### 3. 与列结合使用

算术表达式可以直接使用列名：

```sql
SELECT price * quantity AS total_price,
       price * quantity * 0.1 AS tax
FROM orders
WHERE (price * quantity) > 100;
```

---

### 4. 与常数结合使用

```sql
SELECT salary * 1.05 AS new_salary
FROM employees;
```

常用于 **百分比增长、折扣计算**。

---

### 5. 与函数结合使用

很多数据库提供数学函数，可以在算术表达式中使用：

| 函数                       | 说明       | 示例                               |
| ------------------------ | -------- | -------------------------------- |
| `ROUND(x, n)`            | 四舍五入     | `SELECT ROUND(salary * 1.05, 2)` |
| `CEIL(x)` / `CEILING(x)` | 向上取整     | `SELECT CEIL(salary / 1000)`     |
| `FLOOR(x)`               | 向下取整     | `SELECT FLOOR(salary / 1000)`    |
| `ABS(x)`                 | 绝对值      | `SELECT ABS(-10)`                |
| `POWER(x, y)`            | x 的 y 次方 | `SELECT POWER(2, 3)`             |
| `SQRT(x)`                | 平方根      | `SELECT SQRT(16)`                |
| `MOD(x, y)`              | 取模       | `SELECT MOD(17, 5)`              |

---

### 6. 示例综合

```sql
SELECT 
    price,
    quantity,
    price * quantity AS total,
    price * quantity * 0.1 AS tax,
    ROUND(price * quantity * 1.1, 2) AS total_with_tax
FROM orders
WHERE (price * quantity) > 100;
```

---

💡 **总结**

* SQL 支持 `+ - * / %` 的基本算术运算
* 支持括号控制运算顺序
* 可以与列、常数、数学函数结合
* 聚合函数（如 `SUM(price*quantity)`）可以和算术表达式叠加使用
---

## 十一、SQL 条件判断 & 空值处理函数速

### 1. `IF` 函数

**功能**：根据条件返回不同结果，相当于 SQL 里的三元运算符。

**语法**：

```sql
IF(condition, value_if_true, value_if_false)
```

**示例**：

```sql
SELECT name,
       IF(score >= 60, '及格', '不及格') AS result
FROM students;
```

解释：

* 如果 `score >= 60`，返回 `'及格'`，否则返回 `'不及格'`。

---

### 2. `IFNULL` 函数

**功能**：判断值是否为 `NULL`，如果是 `NULL` 则返回指定默认值。

**语法**：

```sql
IFNULL(expression, default_value)
```

**示例**：

```sql
SELECT name, IFNULL(phone, '未填写') AS phone_number
FROM users;
```

解释：

* 如果 `phone` 是 `NULL`，返回 `'未填写'`；否则返回实际值。

---

### 3.  类似功能的函数/语句

| 函数/语句                                     | 功能                                  | 示例                                                                                                            |
| ----------------------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `COALESCE(expr1, expr2, ...)`             | 返回第一个非 NULL 值                       | `SELECT COALESCE(phone, mobile, '未填写') FROM users;`                                                           |
| `NULLIF(expr1, expr2)`                    | 如果 expr1 = expr2，返回 NULL，否则返回 expr1 | `SELECT NULLIF(score, 0) FROM exams;`                                                                         |
| `CASE ... WHEN ... THEN ... ELSE ... END` | 通用条件表达式，更灵活                         | `sql SELECT name, CASE WHEN score>=90 THEN 'A' WHEN score>=80 THEN 'B' ELSE 'C' END AS grade FROM students; ` |

---

### 4.  总结

1. **`IF`**：类似三元运算符，处理条件判断
2. **`IFNULL`**：处理空值，指定默认值
3. **`COALESCE`**：多值空判断，返回第一个非 NULL
4. **`NULLIF`**：把某个值转换成 NULL（常用于避免除零）
5. **`CASE`**：通用条件表达式，比 `IF` 更灵活，可以处理多个条件

---

非常好 👍 你现在已经在学习 SQL 的 **DML（数据操作语言）** 部分了，这是 SQL 最常用、也最容易考的模块之一。
我来帮你系统地梳理一下：

---

## 十二、 DML（数据操作语言）

### 1. DML（Data Manipulation Language）简介

**DML（数据操作语言）**
是用于 **操作表中数据内容** 的 SQL 语句。
即：**对表中已有的数据进行“增、删、改”操作**。

常见语句包括：

| 操作   | 关键字      | 功能         |
| ---- | -------- | ---------- |
| 增加数据 | `INSERT` | 向表中插入新记录   |
| 修改数据 | `UPDATE` | 修改表中已存在的记录 |
| 删除数据 | `DELETE` | 删除表中已存在的记录 |

---

### 2. `INSERT` —— 插入数据

✅ 语法一：插入完整一行

```sql
INSERT INTO 表名 VALUES (值1, 值2, 值3, ...);
```

示例：

```sql
INSERT INTO students VALUES (1, 'Alice', 20, 'female');
```

> 注意：必须按表的列顺序提供所有值。

---

✅ 语法二：指定列名插入（推荐写法）

```sql
INSERT INTO 表名 (列1, 列2, 列3) VALUES (值1, 值2, 值3);
```

示例：

```sql
INSERT INTO students (name, age) VALUES ('Bob', 22);
```

> ✅ 优点：更安全，避免因表结构变化而出错。

---

✅ 语法三：批量插入多行数据

```sql
INSERT INTO students (name, age, gender)
VALUES 
  ('Cindy', 21, 'female'),
  ('David', 23, 'male'),
  ('Emma', 20, 'female');
```

---

✅ 语法四：通过查询插入数据（复制表数据）

```sql
INSERT INTO new_table (id, name)
SELECT id, name FROM old_table
WHERE age > 20;
```

---

### 3. `UPDATE` —— 修改数据

✅ 基本语法

```sql
UPDATE 表名
SET 列1 = 值1, 列2 = 值2, ...
WHERE 条件;
```

示例：

```sql
UPDATE students
SET age = 21
WHERE name = 'Alice';
```

> ⚠️ **强烈建议带上 `WHERE` 条件**，否则会修改整张表！

---

✅ 支持算术表达式和子查询

```sql
-- 工资加 10%
UPDATE employees
SET salary = salary * 1.1
WHERE department = 'Sales';

-- 根据其他表更新
UPDATE employees e
JOIN department d ON e.dept_id = d.id
SET e.dept_name = d.name;
```

---

### 4. `DELETE` —— 删除数据

✅ 基本语法

```sql
DELETE FROM 表名 WHERE 条件;
```

示例：

```sql
DELETE FROM students WHERE age < 18;
```

> ⚠️ 没有 `WHERE` 会删除整张表所有数据！

---

✅ 清空整张表（推荐用 `TRUNCATE`）

```sql
TRUNCATE TABLE students;
```

| 操作         | 能否回滚   | 是否重置自增ID | 效率  |
| ---------- | ------ | -------- | --- |
| `DELETE`   | ✅ 可回滚  | ❌ 不重置    | 较慢  |
| `TRUNCATE` | ❌ 不可回滚 | ✅ 重置     | 非常快 |

---

### 5. 组合使用

```sql
-- 插入新数据
INSERT INTO orders (id, total) VALUES (1, 500);

-- 更新订单金额
UPDATE orders SET total = total + 100 WHERE id = 1;

-- 删除过期订单
DELETE FROM orders WHERE create_time < '2025-01-01';
```

---

### 6. 小结对比

| 语句         | 功能   | 是否操作整表 | 是否可回滚 |
| ---------- | ---- | ------ | ----- |
| `INSERT`   | 新增数据 | 否      | ✅     |
| `UPDATE`   | 修改数据 | 否      | ✅     |
| `DELETE`   | 删除数据 | 否      | ✅     |
| `TRUNCATE` | 清空表  | 是      | ❌     |

---


