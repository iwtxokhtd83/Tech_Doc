# SQL 与 MySQL 讲义

> 本讲义覆盖标准 SQL 与 MySQL（侧重 InnoDB）两条主线：从基础查询到复杂优化、从事务隔离到索引原理、从 Schema 设计到高可用。每章配"知识点 + 笔试题"。
>
> 约定：示例语法以 MySQL 8.0 为主；涉及到方言差异处会注明。

## 目录

1. [关系模型与 SQL 基础](#1-关系模型与-sql-基础)
2. [DDL：表与 Schema 设计](#2-ddl表与-schema-设计)
3. [DML：增删改查](#3-dml增删改查)
4. [JOIN、子查询与集合运算](#4-join子查询与集合运算)
5. [聚合、分组与窗口函数](#5-聚合分组与窗口函数)
6. [视图、CTE 与存储程序](#6-视图cte-与存储程序)
7. [数据类型与字符集](#7-数据类型与字符集)
8. [约束、索引基础](#8-约束索引基础)
9. [MySQL 架构与存储引擎](#9-mysql-架构与存储引擎)
10. [InnoDB 索引深入](#10-innodb-索引深入)
11. [事务与并发控制](#11-事务与并发控制)
12. [锁机制](#12-锁机制)
13. [执行计划与 SQL 优化](#13-执行计划与-sql-优化)
14. [慢查询与性能排障](#14-慢查询与性能排障)
15. [主从复制与高可用](#15-主从复制与高可用)
16. [分库分表与扩展性](#16-分库分表与扩展性)
17. [备份、恢复与运维](#17-备份恢复与运维)
18. [安全与权限](#18-安全与权限)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 关系模型与 SQL 基础

### 1.1 关系模型概念

- **关系（Relation）**：二维表
- **元组（Tuple）**：一行
- **属性（Attribute）**：一列
- **主键（PK）**：唯一标识一行
- **外键（FK）**：引用其他表的主键
- **候选键 / 超键**：可唯一标识的列组合；主键是候选键之一

### 1.2 SQL 语言分类

| 分类 | 说明 | 典型命令 |
|------|------|----------|
| **DDL** | 数据定义 | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** | 数据操纵 | `INSERT`, `UPDATE`, `DELETE`, `SELECT`（有时归 DQL）|
| **DQL** | 数据查询 | `SELECT` |
| **DCL** | 数据控制 | `GRANT`, `REVOKE` |
| **TCL** | 事务控制 | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

### 1.3 SELECT 执行顺序（逻辑）

```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

**陷阱**：`WHERE` 不能使用 `SELECT` 中的**列别名**（MySQL 例外：SELECT 表达式可在 ORDER BY / GROUP BY 用别名）；`HAVING` 可以用聚合结果。

### 1.4 三值逻辑（NULL）

`NULL` 表示未知，不等于空字符串也不等于 0。任何与 `NULL` 比较的结果是 `NULL`（不是 `TRUE`/`FALSE`）。

```sql
SELECT NULL = NULL;        -- NULL（不是 1）
SELECT NULL <=> NULL;      -- 1（MySQL 的 NULL 安全比较）
SELECT col IS NULL;        -- 正确
WHERE col != 'A'           -- 不会返回 col 为 NULL 的行！
```

### 📝 笔试题 1-1：`SELECT COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)` 的区别？

- `COUNT(*)`：统计**行数**（包含所有列都为 NULL 的行）
- `COUNT(col)`：统计 `col` 非 NULL 的行数
- `COUNT(DISTINCT col)`：去重后非 NULL 值的个数

### 📝 笔试题 1-2：WHERE 和 HAVING 的区别？

- `WHERE`：**分组前**过滤行，不能用聚合
- `HAVING`：**分组后**过滤组，可用聚合结果
- 若无 GROUP BY，HAVING 对整个结果集起作用

---

## 2. DDL：表与 Schema 设计

### 2.1 创建表

```sql
CREATE TABLE users (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  username    VARCHAR(64) NOT NULL,
  email       VARCHAR(255) NOT NULL,
  status      TINYINT NOT NULL DEFAULT 1 COMMENT '0 disabled, 1 active',
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
              ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_username (username),
  UNIQUE KEY uk_email (email),
  KEY idx_status_created (status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 2.2 修改表

```sql
ALTER TABLE users ADD COLUMN nickname VARCHAR(64) NULL AFTER username;
ALTER TABLE users MODIFY COLUMN email VARCHAR(320) NOT NULL;
ALTER TABLE users CHANGE COLUMN username user_name VARCHAR(64) NOT NULL;
ALTER TABLE users DROP COLUMN nickname;
ALTER TABLE users ADD INDEX idx_status (status);
ALTER TABLE users DROP INDEX idx_status;
ALTER TABLE users RENAME TO accounts;
```

**MySQL 8.0 Online DDL** 多数情况下支持 `ALGORITHM=INPLACE, LOCK=NONE`，避免长锁表：

```sql
ALTER TABLE users ADD INDEX idx_email (email), ALGORITHM=INPLACE, LOCK=NONE;
```

大表变更建议用 `pt-online-schema-change` 或 `gh-ost`，避免长时间主从延迟。

### 2.3 删除与清空

```sql
DROP TABLE IF EXISTS users;       -- 删除表结构和数据
TRUNCATE TABLE users;             -- 保留结构，清空数据（DDL，不可回滚，重置自增）
DELETE FROM users;                -- DML，可回滚，不重置自增，速度慢
```

### 2.4 范式与反范式

| 范式 | 要求 |
|------|------|
| 1NF | 每列原子，不可再分 |
| 2NF | 1NF + 非主属性完全依赖主键（消除部分依赖） |
| 3NF | 2NF + 非主属性不传递依赖主键 |
| BCNF | 3NF 更严格版本 |

**实际应用**：OLTP 通常满足 3NF；报表/读多写少可**反范式**（冗余字段、预聚合）换性能。

### 2.5 命名约定（团队参考）

- 表：小写 + 下划线 + 复数，如 `users`、`order_items`
- 主键：`id`
- 外键：`<ref_table>_id`，如 `user_id`
- 索引：`idx_<columns>`；唯一：`uk_<columns>`；外键：`fk_<table>_<columns>`
- 时间：`created_at`、`updated_at`、`deleted_at`
- 布尔：`is_`、`has_`

### 📝 笔试题 2-1：TRUNCATE 和 DELETE 的区别？

| 维度 | TRUNCATE | DELETE |
|------|----------|--------|
| 分类 | DDL | DML |
| 事务 | 不能回滚（隐式提交） | 可回滚 |
| 自增 | 重置 | 不重置 |
| 速度 | 极快（drop + recreate segment） | 按行日志，慢 |
| 触发器 | 不触发 | 触发 |
| 外键约束 | 受约束时失败（InnoDB）| 受约束检查 |

---

## 3. DML：增删改查

### 3.1 INSERT

```sql
-- 单行
INSERT INTO users (username, email) VALUES ('alice', 'a@x.com');

-- 多行（比循环单行快几十倍）
INSERT INTO users (username, email) VALUES
  ('alice','a@x.com'), ('bob','b@x.com');

-- 从查询结果插入
INSERT INTO users_archive SELECT * FROM users WHERE status=0;

-- 主键冲突时更新（Upsert）
INSERT INTO users (id, username, email) VALUES (1, 'alice', 'a@x.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);
-- 8.0.20+ 推荐
INSERT INTO users (id, username, email) VALUES (1, 'alice', 'a@x.com') AS new
ON DUPLICATE KEY UPDATE email = new.email;

-- 忽略冲突
INSERT IGNORE INTO users ...;
```

### 3.2 UPDATE

```sql
UPDATE users SET status = 0 WHERE last_login_at < '2024-01-01';

-- 多列更新
UPDATE users SET status = 0, updated_at = NOW() WHERE id = 42;

-- 多表 UPDATE
UPDATE orders o JOIN users u ON o.user_id = u.id
SET o.status = 'frozen'
WHERE u.status = 0;

-- 限制行数（安全刹车）
UPDATE logs SET archived = 1 WHERE created_at < '2024-01-01' LIMIT 1000;
```

**强制加 WHERE**：生产环境可配 `sql_safe_updates=1`，防止无 WHERE 的全表更新。

### 3.3 DELETE

```sql
DELETE FROM users WHERE status = 0;
DELETE FROM users WHERE status = 0 LIMIT 1000;  -- 分批

-- 多表 DELETE
DELETE o FROM orders o JOIN users u ON o.user_id = u.id WHERE u.status = 0;

-- 软删除（推荐）
UPDATE users SET deleted_at = NOW() WHERE id = 42;
```

### 3.4 SELECT 基础

```sql
SELECT id, username FROM users
WHERE status = 1 AND created_at >= '2025-01-01'
ORDER BY created_at DESC
LIMIT 10 OFFSET 20;
```

**注意**：

- `SELECT *` 不利于索引覆盖与网络传输，上生产应**显式列出字段**
- `LIMIT` 配合 `ORDER BY` 保证确定性；否则结果顺序依赖执行路径

### 📝 笔试题 3-1：`INSERT ... ON DUPLICATE KEY UPDATE` 与 `REPLACE INTO` 的区别？

- `ON DUPLICATE KEY UPDATE`：冲突时**更新**原行；自增主键不变
- `REPLACE INTO`：冲突时**先删后插**；自增主键**会变**；外键引用可能级联删，风险更大

推荐使用 `ON DUPLICATE KEY UPDATE`。

---

## 4. JOIN、子查询与集合运算

### 4.1 JOIN 类型

```sql
-- 内连接（INNER JOIN）：两边都必须匹配
SELECT u.*, o.id AS order_id
FROM users u JOIN orders o ON o.user_id = u.id;

-- 左外连接（LEFT JOIN）：保留左表所有行
SELECT u.*, o.id AS order_id
FROM users u LEFT JOIN orders o ON o.user_id = u.id;

-- 右外连接：保留右表（少用，一般调换表序用 LEFT）
-- 全外连接：MySQL 不支持，需 UNION 模拟
SELECT * FROM A LEFT JOIN B ON ... 
UNION
SELECT * FROM A RIGHT JOIN B ON ...;

-- 交叉连接（笛卡尔积）
SELECT * FROM A CROSS JOIN B;     -- 等价于 FROM A, B（无条件）
```

### 4.2 JOIN ON vs WHERE

```sql
-- 正确：过滤在 ON 里，保证左表全部保留
SELECT u.*, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'paid';

-- 错误：过滤写在 WHERE 会把 LEFT JOIN 降级成 INNER
SELECT u.*, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid';      -- 没有订单的用户被过滤掉了！
```

### 4.3 自连接

```sql
-- 每个员工的经理姓名
SELECT e.name AS emp, m.name AS mgr
FROM employees e LEFT JOIN employees m ON e.manager_id = m.id;
```

### 4.4 子查询

```sql
-- 标量子查询
SELECT id, (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_cnt
FROM users u;

-- IN 子查询
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE status = 'paid');

-- EXISTS（更灵活，行数多时常比 IN 快）
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'paid');

-- 派生表（FROM 子查询）
SELECT t.user_id, t.total
FROM (SELECT user_id, SUM(amount) AS total FROM orders GROUP BY user_id) t
WHERE t.total > 1000;
```

### 4.5 集合运算

```sql
-- 并集去重
SELECT id FROM a UNION SELECT id FROM b;
-- 并集不去重
SELECT id FROM a UNION ALL SELECT id FROM b;

-- 交集（MySQL 8.0.31+ 支持 INTERSECT）
-- 差集（EXCEPT，8.0.31+）
```

### 📝 笔试题 4-1：LEFT JOIN 后在 WHERE 里写右表条件会发生什么？

会把 `LEFT JOIN` 降级成 `INNER JOIN`——右表不匹配的行右列为 NULL，再被 WHERE 过滤掉。**想保留左表则条件写在 ON 上**；WHERE 中可用 `IS NULL` 专门找"左表独有"的行。

### 📝 笔试题 4-2：IN 和 EXISTS 哪个快？

MySQL 8.0 优化器会基于统计信息选择，现代版本基本等价。历史经验：

- 内表小：`IN` 常见转半连接
- 外表小、内表大：`EXISTS` 可能更好
- 不要依赖口诀，**以 `EXPLAIN` 为准**

---

## 5. 聚合、分组与窗口函数

### 5.1 聚合函数

```sql
SELECT COUNT(*), COUNT(DISTINCT user_id), SUM(amount), AVG(amount),
       MIN(amount), MAX(amount), GROUP_CONCAT(tag SEPARATOR ',')
FROM orders;
```

### 5.2 GROUP BY + HAVING

```sql
SELECT user_id, COUNT(*) cnt, SUM(amount) total
FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING total > 1000
ORDER BY total DESC;
```

MySQL 早期允许 `SELECT` 非 GROUP 列（取"任意值"）；8.0 默认开启 `ONLY_FULL_GROUP_BY`，要求 `SELECT` 列要么在 GROUP BY 中，要么是聚合。

### 5.3 WITH ROLLUP

```sql
SELECT category, SUM(amount) FROM orders
GROUP BY category WITH ROLLUP;
-- 会额外多一行总计（category 为 NULL）
```

### 5.4 窗口函数（8.0+）

不分组，保留原行，按窗口计算：

```sql
-- 排名
SELECT id, user_id, amount,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn,
  RANK()       OVER (PARTITION BY user_id ORDER BY amount DESC)     AS rk,
  DENSE_RANK() OVER (PARTITION BY user_id ORDER BY amount DESC)     AS drk
FROM orders;

-- 累计和
SELECT id, created_at, amount,
  SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;

-- 前后行取值
SELECT id, amount,
  LAG(amount, 1)  OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount,
  LEAD(amount, 1) OVER (PARTITION BY user_id ORDER BY created_at) AS next_amount
FROM orders;

-- 每用户最新一笔
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) rn
  FROM orders
) t WHERE rn = 1;
```

### 5.5 ROW_NUMBER vs RANK vs DENSE_RANK

```
amount  ROW_NUMBER  RANK  DENSE_RANK
100          1       1        1
100          2       1        1
80           3       3        2
50           4       4        3
```

### 📝 笔试题 5-1：取每个用户最新一条订单？

**方法 A**（窗口函数，推荐 8.0+）：

```sql
SELECT id, user_id, created_at
FROM (
  SELECT id, user_id, created_at,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) rn
  FROM orders
) t
WHERE rn = 1;
```

**方法 B**（关联子查询，老版本）：

```sql
SELECT o.*
FROM orders o
WHERE o.created_at = (SELECT MAX(created_at) FROM orders o2 WHERE o2.user_id = o.user_id);
```

---

## 6. 视图、CTE 与存储程序

### 6.1 视图

```sql
CREATE OR REPLACE VIEW v_active_users AS
SELECT id, username FROM users WHERE status = 1;

SELECT * FROM v_active_users WHERE username LIKE 'a%';
```

- **优点**：封装复杂查询、权限控制、向前兼容
- **限制**：部分视图不可更新（含聚合、JOIN、DISTINCT 等）；性能取决于底层查询，不会自动"物化"

### 6.2 CTE（Common Table Expression，8.0+）

```sql
-- 简单 CTE
WITH recent AS (
  SELECT * FROM orders WHERE created_at >= CURDATE() - INTERVAL 7 DAY
)
SELECT user_id, COUNT(*) FROM recent GROUP BY user_id;

-- 递归 CTE：查组织树
WITH RECURSIVE dept_tree AS (
  SELECT id, name, parent_id, 1 AS level FROM departments WHERE parent_id IS NULL
  UNION ALL
  SELECT d.id, d.name, d.parent_id, t.level + 1
  FROM departments d JOIN dept_tree t ON d.parent_id = t.id
)
SELECT * FROM dept_tree ORDER BY level, name;
```

### 6.3 存储过程与函数

```sql
DELIMITER //
CREATE PROCEDURE archive_old_orders(IN days INT)
BEGIN
  DECLARE affected INT DEFAULT 0;
  START TRANSACTION;
    INSERT INTO orders_archive
      SELECT * FROM orders WHERE created_at < NOW() - INTERVAL days DAY;
    SET affected = ROW_COUNT();
    DELETE FROM orders WHERE created_at < NOW() - INTERVAL days DAY;
  COMMIT;
  SELECT affected;
END //
DELIMITER ;

CALL archive_old_orders(90);
```

### 6.4 触发器

```sql
CREATE TRIGGER trg_users_audit
AFTER UPDATE ON users
FOR EACH ROW
INSERT INTO users_audit(user_id, old_status, new_status, changed_at)
VALUES (OLD.id, OLD.status, NEW.status, NOW());
```

**慎用**：调试困难、隐式副作用、分库分表环境不友好，能用应用层就别用触发器。

### 6.5 事件调度器

```sql
SET GLOBAL event_scheduler = ON;

CREATE EVENT ev_cleanup_sessions
ON SCHEDULE EVERY 1 HOUR
DO DELETE FROM sessions WHERE expire_at < NOW();
```

### 📝 笔试题 6-1：视图会提升性能吗？

通常**不会**。标准视图每次查询都展开为底层 SQL 执行。真正"加速"需要**物化视图**（MySQL 无原生；可通过定时任务 + 表模拟）。视图的价值在**封装与权限**，不在性能。

---

## 7. 数据类型与字符集

### 7.1 数值

| 类型 | 字节 | 范围（有符号） |
|------|------|----------------|
| `TINYINT` | 1 | -128 ~ 127 |
| `SMALLINT` | 2 | -32768 ~ 32767 |
| `MEDIUMINT` | 3 | -2²³ ~ 2²³-1 |
| `INT` | 4 | -2³¹ ~ 2³¹-1 |
| `BIGINT` | 8 | -2⁶³ ~ 2⁶³-1 |
| `DECIMAL(M,D)` | 变长 | 精确小数，金额首选 |
| `FLOAT / DOUBLE` | 4 / 8 | IEEE 754，**不精确** |

注意：`INT(11)` 的 `11` 只是**显示宽度**，与存储大小无关；MySQL 8.0 已弃用。

### 7.2 字符串

| 类型 | 说明 | 适用 |
|------|------|------|
| `CHAR(N)` | 定长，填充空格 | 定长固定内容（如 MD5 的 32 位）|
| `VARCHAR(N)` | 变长，最多 65535 字节（整行限制） | 大部分短字符串 |
| `TEXT` / `MEDIUMTEXT` / `LONGTEXT` | 大文本，不可有默认值 | 长文 |
| `BLOB` / `MEDIUMBLOB` / `LONGBLOB` | 二进制 | 文件、图片 |
| `JSON` | 原生 JSON，可用函数查询 | 半结构化 |
| `ENUM('A','B')` | 枚举 | 小状态，慎用（ALTER 不便） |
| `SET('a','b','c')` | 多值集合 | 同上，少用 |

`VARCHAR` 长度按**字符数**定义，但**受行内总字节数**限制；`utf8mb4` 单字符最多 4 字节。

### 7.3 日期时间

| 类型 | 范围 | 说明 |
|------|------|------|
| `DATE` | 1000-01-01 ~ 9999-12-31 | 仅日期 |
| `TIME` | -838:59:59 ~ 838:59:59 | 时段 |
| `DATETIME` | 1000-01-01 ~ 9999-12-31 | 时间点，8 字节，不随时区变化 |
| `TIMESTAMP` | 1970-2038 | 4 字节，存储为 UTC，会按会话时区转换 |
| `YEAR` | 1901-2155 | 2 字节 |

**坑**：`TIMESTAMP` 的 2038 问题；跨时区系统建议用 `DATETIME` + 应用层统一 UTC。

### 7.4 字符集与排序规则

```sql
SHOW CHARACTER SET;
SHOW COLLATION LIKE 'utf8mb4%';
```

- **首选** `utf8mb4`（真正的 UTF-8，支持 emoji）
- MySQL 历史上的 `utf8` 实为 `utf8mb3`（最多 3 字节），已弃用
- 默认排序规则：
  - 8.0 默认 `utf8mb4_0900_ai_ci`（Unicode 9.0，accent/case insensitive）
  - `_bin` 区分大小写与重音
  - 业务场景选对 collation 很重要（如区分大小写的登录名）

### 7.5 JSON 类型

```sql
CREATE TABLE events (
  id BIGINT PRIMARY KEY,
  payload JSON
);

INSERT INTO events VALUES (1, '{"user":{"id":42,"name":"Alice"},"tags":["a","b"]}');

-- 提取
SELECT payload->>'$.user.name' FROM events WHERE id = 1;   -- "Alice"（去引号）
SELECT JSON_EXTRACT(payload, '$.tags[0]') FROM events;

-- 索引：用生成列 + 索引
ALTER TABLE events
  ADD COLUMN user_id INT AS (CAST(payload->>'$.user.id' AS UNSIGNED)) STORED,
  ADD INDEX idx_user_id (user_id);
```

### 📝 笔试题 7-1：金额字段用什么类型？

**`DECIMAL(M, D)`**，如 `DECIMAL(18, 2)`。浮点 `FLOAT/DOUBLE` 精度不够，会出现 `0.1 + 0.2 != 0.3`。高频业务也可用 **`BIGINT`** 存最小单位（分/厘），应用层格式化。

### 📝 笔试题 7-2：为什么别用 MySQL 历史的 `utf8`？

MySQL 历史 `utf8` 实为 `utf8mb3`，最多 3 字节，无法表示辅助平面字符（emoji 😂 等）。应**统一用 `utf8mb4`**。

---

## 8. 约束、索引基础

### 8.1 常见约束

- `PRIMARY KEY`：主键，非空+唯一
- `UNIQUE`：唯一
- `NOT NULL`
- `DEFAULT`
- `CHECK`（MySQL 8.0.16+ 真正生效）
- `FOREIGN KEY`：外键
- `AUTO_INCREMENT`：自增

### 8.2 外键

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE RESTRICT ON UPDATE CASCADE
);
```

生产环境**互联网业务多数不使用外键**：跨库不可用、分库分表不友好、写入性能损耗、业务上由应用/消息保证一致性。金融/传统行业常保留。

### 8.3 索引分类

| 索引类型 | 说明 |
|----------|------|
| **B+Tree**（默认） | 范围、排序、等值都支持 |
| **哈希索引** | Memory 引擎；InnoDB 有自适应哈希（内部优化） |
| **全文索引** | `FULLTEXT`，对长文本分词 |
| **空间索引** | `SPATIAL`，R-Tree |
| **前缀索引** | 对字符串前 N 字符建索引，省空间 |
| **联合索引** | 多列组合，注意顺序 |
| **唯一索引** | 值唯一 |
| **覆盖索引** | 查询所需列完全在索引中，无需回表 |

### 8.4 创建索引

```sql
CREATE INDEX idx_status_created ON orders (status, created_at);
CREATE UNIQUE INDEX uk_email ON users (email);

-- 前缀索引：节省空间，但不能覆盖 / 排序
CREATE INDEX idx_addr ON users (address(20));

-- 函数索引（8.0.13+）
CREATE INDEX idx_lower_name ON users ((LOWER(name)));
```

### 8.5 最左前缀原则

对于联合索引 `(a, b, c)`：

- ✅ 可用：`WHERE a=? AND b=? AND c=?`、`WHERE a=? AND b=?`、`WHERE a=?`
- ✅ 可用：`WHERE a=? AND c=?`（用到 `a`，`c` 未参与索引定位）
- ✅ 范围到此为止：`WHERE a=? AND b>? AND c=?`（`b` 范围后，`c` 不再走索引等值）
- ❌ 无法使用：`WHERE b=?`、`WHERE c=?`

**排序**：`ORDER BY a, b, c` 可用该索引避免排序；`ORDER BY b` 不行。

### 📝 笔试题 8-1：联合索引 `(a,b,c)`，`WHERE b=1 AND a=2 AND c=3` 能走索引吗？

**能**。优化器会重排条件顺序；关键是**条件集合**覆盖了索引的最左字段，而不是写法顺序。

### 📝 笔试题 8-2：`VARCHAR(1000)` 的列如何高效加索引？

用**前缀索引**：`CREATE INDEX idx_col ON t (col(32));`，并评估**区分度**（`SELECT COUNT(DISTINCT LEFT(col, 32)) / COUNT(*)`）以决定前缀长度。代价：不能用作覆盖索引，不能用于 ORDER BY。

---

## 9. MySQL 架构与存储引擎

### 9.1 分层架构

```
连接层         (认证、连接管理、线程池)
  │
Server 层      (解析器、优化器、执行器、缓存*)
  │
存储引擎层     (InnoDB / MyISAM / Memory / Archive / ...)
  │
文件系统       (数据文件、日志、表空间)
```

*Query Cache 在 8.0 已被移除。

### 9.2 一次查询的旅程

1. **连接器**：握手、鉴权、权限检查
2. **解析器**：语法/语义分析，生成解析树
3. **预处理**：检查表名、列名、权限
4. **优化器**：生成执行计划（成本模型）
5. **执行器**：调用存储引擎 API
6. **存储引擎**：返回数据；Server 层汇总返回客户端

### 9.3 常见存储引擎对比

| 引擎 | 事务 | 行锁 | 外键 | 崩溃恢复 | 典型用途 |
|------|------|------|------|----------|----------|
| **InnoDB** | ✅ | ✅ | ✅ | ✅（Redo Log） | 默认，OLTP |
| **MyISAM** | ❌ | 表锁 | ❌ | ❌ | 只读报表、全文老版 |
| **Memory** | ❌ | 表锁 | ❌ | 重启丢 | 临时缓存 |
| **Archive** | ❌ | 行锁 | ❌ | 重启丢 | 归档 |
| **CSV** | ❌ | 表锁 | ❌ | — | 文本导入 |

99% 场景用 **InnoDB**。

### 9.4 InnoDB 关键文件

- 表空间：`ibdata1`（系统表空间）、`*.ibd`（独立表空间，`innodb_file_per_table=ON`）
- **Redo Log**：`ib_logfile0/1`，WAL 保证 crash safe
- **Undo Log**：存于系统或独立 undo 表空间，支持事务回滚与 MVCC
- **Doublewrite Buffer**：防页面写入撕裂
- `binlog`：**Server 层**归档日志，用于复制与 PITR（与存储引擎无关）

### 📝 笔试题 9-1：InnoDB 与 MyISAM 的主要区别？

- 事务：InnoDB 支持，MyISAM 不支持
- 锁粒度：InnoDB 行锁，MyISAM 表锁
- 崩溃恢复：InnoDB 有 redo log，MyISAM 需 `myisamchk`
- 索引：InnoDB 主键聚簇（数据与主键在一起），MyISAM 索引与数据分离
- 外键：InnoDB 支持
- 全文：MyISAM 早期支持，InnoDB 5.6+ 也支持

---

## 10. InnoDB 索引深入

### 10.1 聚簇索引

InnoDB 的**主键索引**叶子节点直接存**整行数据**。选择：

- 有主键：用主键
- 无主键：用第一个非空唯一索引
- 都没有：内部生成 6 字节的 `ROW_ID`

**推论**：

- 主键顺序插入（自增 / snowflake 递增）避免页分裂
- 主键太长 → 所有二级索引也变大（二级索引叶子存主键值）

### 10.2 二级索引与回表

二级索引叶子存 `(索引列, 主键)`。通过二级索引查到主键后再回主键索引取行 = **回表**。

**覆盖索引**：所需列全在二级索引中，无需回表：

```sql
-- 有索引 idx_status_created(status, created_at)
SELECT status, created_at FROM orders WHERE status = 'paid';   -- 覆盖索引
SELECT *                  FROM orders WHERE status = 'paid';   -- 需回表
```

`EXPLAIN` 的 `Extra` 列出现 `Using index` 即覆盖。

### 10.3 索引下推（ICP，Index Condition Pushdown）

MySQL 5.6+。联合索引的非前缀部分条件可在存储引擎层过滤，**减少回表次数**：

```sql
-- idx(name, age)
SELECT * FROM users WHERE name LIKE 'A%' AND age = 30;
```

`age = 30` 的判断下推到引擎层，仅匹配的才回表。`EXPLAIN` 的 `Extra` 显示 `Using index condition`。

### 10.4 失效场景

- **函数或表达式作用在索引列**：`WHERE DATE(created_at)=?`、`WHERE a+1=5`
- **隐式类型转换**：`WHERE phone = 13800138000`（phone 是 VARCHAR）
- **字符集不同的 JOIN**：驱动表列做 convert
- **`LIKE '%xxx'`** 前缀通配
- **`OR` 两边一边无索引**（8.0+ 有 index merge 改善，但仍建议避免）
- **`NOT IN` / `<>`**：通常全扫
- **`IS NULL`**：看数据分布与版本
- **排序列与索引顺序/方向不一致**（8.0 前降序索引支持差）

### 10.5 最佳实践

- **主键用自增 BIGINT**：小且单调
- **高选择性列在联合索引前面**
- **最常用于范围查询的列放最后**（范围之后索引失效）
- **避免过多索引**：每个索引都有写入开销（B+Tree 维护 + 二级索引同步）
- **在线变更**：大表加索引用 `pt-osc` / `gh-ost`
- **定期分析**：`ANALYZE TABLE t;` 让优化器拿到准确基数

### 📝 笔试题 10-1：什么是"回表"？如何避免？

二级索引找到主键后再去主键索引取完整行的过程叫"回表"。避免方式：

- **覆盖索引**：把查询需要的列都加入联合索引
- **减少 `SELECT *`**，只取需要的列

### 📝 笔试题 10-2：为什么 InnoDB 推荐自增主键？

- 顺序插入，B+Tree 只在末端分裂，**避免页分裂与碎片**
- 主键短小，二级索引体积小
- 顺序写入对磁盘/SSD 友好，更好利用 buffer pool

---

## 11. 事务与并发控制

### 11.1 ACID

- **A** Atomicity：原子，要么全成要么全无
- **C** Consistency：约束一致
- **I** Isolation：并发隔离
- **D** Durability：持久化

InnoDB 通过 **Redo Log（WAL）** 保证 D，**Undo Log** 保证 A，锁和 MVCC 保证 I。

### 11.2 隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|-----------|------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED (RC) | ❌ | ✅ | ✅ |
| REPEATABLE READ (RR，**MySQL 默认**) | ❌ | ❌ | 基本❌（InnoDB 用 Next-Key Lock 消除） |
| SERIALIZABLE | ❌ | ❌ | ❌ |

### 11.3 三大并发问题

- **脏读**：读到其他事务未提交的修改
- **不可重复读**：同一事务内两次读同一行，值不同（其他事务 UPDATE）
- **幻读**：同一事务内同一范围两次查询，行数不同（其他事务 INSERT/DELETE）

### 11.4 MVCC（多版本并发控制）

InnoDB 为每行维护版本链：

- 每行有隐藏列 `DB_TRX_ID`（最近修改事务 ID）、`DB_ROLL_PTR`（指向 Undo Log 的前版本）
- 事务启动时生成 **Read View**，决定哪些版本可见
- **一致性非锁定读**（普通 `SELECT`）读快照，不加锁
- **当前读**（`SELECT ... FOR UPDATE/LOCK IN SHARE MODE`、`UPDATE`、`DELETE`）读最新版本并加锁

**RC vs RR 的 Read View**：

- RC：**每次** SELECT 新建 Read View → 不可重复读
- RR：**事务开始第一次** SELECT 建立，整个事务复用 → 可重复读

### 11.5 事务控制

```sql
START TRANSACTION;                    -- 或 BEGIN
-- SQL ...
COMMIT;                               -- 提交
ROLLBACK;                             -- 回滚

SAVEPOINT sp1;
-- SQL ...
ROLLBACK TO SAVEPOINT sp1;
RELEASE SAVEPOINT sp1;

-- 设置隔离
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 自动提交
SHOW VARIABLES LIKE 'autocommit';
SET autocommit = 0;
```

### 11.6 幻读与 Next-Key Lock

`RR` 下 InnoDB 用 **Next-Key Lock = 行锁 + 间隙锁（Gap Lock）** 锁住范围，防止插入：

```sql
-- 假设 id 已有 10, 20, 30
SELECT * FROM t WHERE id BETWEEN 15 AND 25 FOR UPDATE;
-- 锁住间隙 (10, 20] 和 (20, 30]，其他事务无法插入 18、22、29
```

### 📝 笔试题 11-1：MySQL 默认隔离级别是什么？为什么不是 RC？

默认 **REPEATABLE READ**。历史原因：早期 `statement` 格式的 binlog 在 RC 下主从数据可能不一致（同一条 UPDATE 在主备读到快照不同），RR + Gap Lock 才能保证。`row` 格式 binlog 后 RC 也能正确复制，很多大厂（阿里等）**线上改为 RC** 减少间隙锁、提升并发。

### 📝 笔试题 11-2：RR 下真的没有幻读吗？

**快照读**（普通 SELECT）不会幻读，Read View 固定。**当前读**（`SELECT ... FOR UPDATE`、`UPDATE` 等）配合 Next-Key Lock 阻止新插入，所以也基本无幻读。但极端 corner case（同事务内先快照读后当前读）仍可能见到"新"行，工程上可认为 InnoDB 的 RR 消除了幻读。

---

## 12. 锁机制

### 12.1 全局锁 / 表级锁 / 行级锁

- **全局锁**：`FLUSH TABLES WITH READ LOCK;`（用于一致性备份）
- **表锁**：`LOCK TABLES t READ/WRITE;`
- **元数据锁（MDL）**：自动加，DDL 与 DML 互斥
- **意向锁**（`IS`、`IX`）：表级，方便判断是否与行锁冲突
- **行锁**：`Record Lock`、`Gap Lock`、`Next-Key Lock`

### 12.2 共享锁 vs 排它锁

```sql
SELECT * FROM t WHERE id=1 LOCK IN SHARE MODE;    -- S 锁（8.0 推荐 FOR SHARE）
SELECT * FROM t WHERE id=1 FOR UPDATE;            -- X 锁
```

| | S | X |
|-|---|---|
| S | ✅ | ❌ |
| X | ❌ | ❌ |

### 12.3 InnoDB 行锁要点

- **行锁加在索引上**：没走索引 → 退化成**表锁**（扫全表加锁）
- 普通 UPDATE/DELETE 会自动加 X 锁
- 插入意向锁（Insert Intention Lock）：等待间隙释放

### 12.4 MDL（元数据锁）

- 读 DML 持 MDL 读锁；DDL 需 MDL 写锁
- 长事务 + DDL 的组合可能导致**连环阻塞**：一个未提交的小事务挡住 DDL，DDL 又挡住后续所有读写
- 生产上 DDL 前先 `SHOW PROCESSLIST` 看长事务

### 12.5 死锁

```sql
-- 事务1
UPDATE a SET x=1 WHERE id=1;
UPDATE b SET y=1 WHERE id=1;
-- 事务2（反序）
UPDATE b SET y=2 WHERE id=1;
UPDATE a SET x=2 WHERE id=1;
-- → 死锁
```

InnoDB 检测到死锁会**回滚代价较小的一个**并报 `Deadlock found`。

排查：`SHOW ENGINE INNODB STATUS\G` 的 `LATEST DETECTED DEADLOCK` 段。

**预防**：

- 固定加锁顺序
- 尽量短事务、小事务
- 对热点行加索引，避免锁表
- 必要时对事务设超时：`innodb_lock_wait_timeout`

### 📝 笔试题 12-1：`UPDATE t SET x=1 WHERE name='a'`，`name` 无索引会怎样？

会扫全表，**对所有扫到的行**（实际 InnoDB 早期版本对全部行加锁，8.0+ 会释放非匹配行的锁）加 X 锁，等价于表锁级别的阻塞，严重影响并发。**生产必须对 WHERE 列建索引**。

---

## 13. 执行计划与 SQL 优化

### 13.1 EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'paid';
EXPLAIN FORMAT=JSON ...;
EXPLAIN ANALYZE ...;    -- 8.0.18+，真实执行并返回耗时
```

### 13.2 关键列

| 列 | 含义 |
|----|------|
| `id` | 查询序号；越大越先执行 |
| `select_type` | SIMPLE/PRIMARY/SUBQUERY/DERIVED 等 |
| `table` | 表名 |
| `type` | 访问类型（**性能核心**） |
| `possible_keys` | 可能使用的索引 |
| `key` | 实际使用的索引 |
| `key_len` | 使用的索引长度（能判断用了联合索引的几列）|
| `ref` | 与索引比较的列或常量 |
| `rows` | 估计扫描行数 |
| `filtered` | 过滤后保留百分比 |
| `Extra` | 补充信息 |

### 13.3 访问类型（type）从好到差

```
system > const > eq_ref > ref > range > index > ALL
```

- `const`：主键/唯一索引等值
- `ref`：非唯一索引等值
- `range`：范围扫描
- `index`：索引全扫（仍比表扫好）
- `ALL`：**全表扫描**（红灯）

### 13.4 Extra 常见值

- `Using index`：**覆盖索引**，好
- `Using where`：用 WHERE 过滤
- `Using index condition`：索引下推
- `Using filesort`：**需要额外排序**，可能要加索引
- `Using temporary`：**用了临时表**，需关注
- `Using join buffer`：未走索引的 JOIN

### 13.5 常见优化手段

1. **加合适的索引**：尤其 WHERE、JOIN、ORDER BY 列
2. **避免 `SELECT *`**
3. **消除索引失效**：函数、类型转换、前导模糊
4. **拆分大查询**：分批、分页、异步
5. **改写 SQL**：子查询改 JOIN，`NOT IN` 改 `LEFT JOIN IS NULL`
6. **分页优化**：
   ```sql
   -- 差：OFFSET 大时慢
   SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
   -- 好：延迟关联
   SELECT o.* FROM orders o JOIN (
     SELECT id FROM orders ORDER BY id LIMIT 1000000, 20
   ) t ON o.id = t.id;
   -- 更好：键集
   SELECT * FROM orders WHERE id > ? ORDER BY id LIMIT 20;
   ```
7. **COUNT 优化**：
   - `COUNT(*)` 在 InnoDB 会扫索引（一般选最小的非空索引）
   - 精确计数贵时改用**缓存**或 `SHOW TABLE STATUS`（近似）
8. **使用 HINT**：`/*+ INDEX(t idx_xx) */`、`FORCE INDEX`、`STRAIGHT_JOIN`

### 13.6 优化器统计信息

- 执行计划依赖表统计（`information_schema.STATISTICS`）
- `ANALYZE TABLE t;` 手动更新
- `innodb_stats_persistent` 持久化统计

### 📝 笔试题 13-1：`EXPLAIN` 里 `type=ALL` 且 `rows` 很大说明什么？

**全表扫描**，未使用索引或没有合适索引。排查：

1. 检查 WHERE 条件是否命中索引
2. 是否发生隐式类型转换（数据库字段 VARCHAR，参数传数值）
3. 统计信息是否过旧，`ANALYZE TABLE` 试试
4. 是否需要加新索引或调整已有索引

---

## 14. 慢查询与性能排障

### 14.1 慢查询日志

```ini
# my.cnf
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
```

分析工具：

- `mysqldumpslow -s t /var/log/mysql/slow.log`
- `pt-query-digest slow.log`（推荐）

### 14.2 performance_schema / sys

```sql
-- 当前运行的 SQL
SELECT * FROM sys.processlist ORDER BY time DESC LIMIT 10;

-- 最耗时 SQL
SELECT * FROM sys.statement_analysis LIMIT 20;

-- 没走索引的 SQL
SELECT * FROM sys.statements_with_full_table_scans;

-- 锁等待
SELECT * FROM sys.innodb_lock_waits;
```

### 14.3 SHOW 命令

```sql
SHOW PROCESSLIST;                     -- 全部线程
SHOW FULL PROCESSLIST;                -- 完整 SQL
SHOW ENGINE INNODB STATUS\G           -- 死锁、长事务、缓冲池
SHOW GLOBAL STATUS LIKE 'Threads%';
SHOW VARIABLES LIKE 'innodb_buffer%';
SHOW INDEX FROM orders;
```

### 14.4 常见性能指标

- **QPS / TPS**
- **缓冲池命中率**：`Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests`（应接近 0）
- **慢查询数 / 比例**
- **活跃连接 / 最大连接**
- **主从延迟**：`SHOW SLAVE STATUS\G` 的 `Seconds_Behind_Master`

### 14.5 排障思路

1. 收到告警 → `SHOW PROCESSLIST` 看热点 SQL / 长事务
2. 慢的 SQL → `EXPLAIN` 看执行计划
3. 锁等待 → `information_schema.INNODB_TRX` + `INNODB_LOCKS`
4. 死锁 → `SHOW ENGINE INNODB STATUS`
5. 资源：CPU 高（算）、IO 高（磁盘）、内存紧（swap）
6. 历史：慢日志聚合、分析同类 SQL

### 📝 笔试题 14-1：生产 CPU 突然 100%，几步定位到问题 SQL？

```sql
SELECT * FROM sys.processlist WHERE state <> '' ORDER BY time DESC LIMIT 10;
-- 或
SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started LIMIT 10;
```

拿到 SQL 后 `EXPLAIN`，定位索引/写法问题；必要时 `KILL <thread_id>` 终止。

---

## 15. 主从复制与高可用

### 15.1 binlog 格式

- **STATEMENT**：记录 SQL 语句，紧凑但部分函数不安全（`NOW()` 等）
- **ROW**：记录行变更，安全但日志大（**生产首选**）
- **MIXED**：混合，MySQL 自行判断

### 15.2 异步复制流程

```
Master                    Slave
 ┌─────────────┐         ┌──────────────┐
 │ Binlog Dump │ ─────── │ IO Thread    │ → relay-log
 └─────────────┘         └──────────────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ SQL Thread   │ → apply
                         └──────────────┘
```

1. Master 写 binlog
2. Slave IO 线程拉 binlog，落为 relay-log
3. Slave SQL 线程读 relay-log 重放

### 15.3 复制模式

- **异步复制**（默认）：可能丢数据
- **半同步（Semi-Sync）**：至少一个 Slave 收到并 ACK 后 Master 才返回客户端
- **组复制（Group Replication）**：基于 Paxos 变体的多主/单主组，8.0 推荐
- **InnoDB Cluster**：GR + MySQL Router + MySQL Shell

### 15.4 GTID vs 基于位点

- **GTID**：全局事务 ID，故障切换简单，**生产推荐**
- **Binlog + Position**：传统方式，切主繁琐

### 15.5 主从延迟

原因：

- 大事务（单 SQL 改百万行）
- 从库单线程重放（5.6 前）；`slave_parallel_workers` + `LOGICAL_CLOCK` 并行
- 从库硬件差、IO 瓶颈
- DDL 阻塞

诊断：`Seconds_Behind_Master`（不完全准）、`pt-heartbeat` 更精确。

### 15.6 读写分离

- 应用层路由（Sharding-JDBC、MyCat、ProxySQL）
- 强一致读仍走主
- 从库可能延迟，避免"写后读"跑到从库

### 15.7 高可用方案

- **MHA**（经典，5.x 流行）
- **Orchestrator**
- **MGR + MySQL Router**（8.0 推荐）
- 云厂商 RDS 自带主备与自动切换

### 📝 笔试题 15-1：主从延迟如何排查？

- 确认延迟：`SHOW SLAVE STATUS\G`、`pt-heartbeat`
- 是否大事务：binlog 里找大事务
- 单线程 vs 并行：`SHOW VARIABLES LIKE 'slave_parallel_%'`
- 硬件：从库 IO/CPU 是否瓶颈
- 大查询：从库慢查询日志
- DDL 阻塞：看 processlist
- 临时方案：跳过非关键 SQL 要极其谨慎（`SET GTID_NEXT`）

---

## 16. 分库分表与扩展性

### 16.1 垂直拆分

- **垂直分库**：不同业务拆到不同库（用户库、订单库、商品库）
- **垂直分表**：一张大表拆成几张（冷热、大字段分离）

### 16.2 水平拆分

把一张表按规则切到多张"分片表"，常见策略：

- **取模**：`id % N`
- **范围**：按日期、ID 区间
- **一致性哈希**：扩容影响小
- **基因法**：用户 ID 后几位作为订单分片键，保证按用户查询时不跨片

### 16.3 中间件

- **Sharding-JDBC**（ShardingSphere）：嵌入式
- **MyCat / DBLE**：Proxy
- **Vitess**：YouTube 开源，TiDB 之外的 NewSQL 候选
- **TiDB / OceanBase / PolarDB**：分布式 NewSQL，天然水平扩展

### 16.4 分片带来的问题

| 问题 | 方案 |
|------|------|
| 跨片 JOIN | 广播小表、应用层聚合、禁止 |
| 跨片事务 | 本地事务 + 最终一致（MQ）、TCC、Seata |
| 全局唯一 ID | Snowflake、号段、UUID |
| 分页 | 归并排序、限制跳页 |
| 排序 / 聚合 | 汇总层（数仓） |
| 扩容 | 预分片、一致性哈希 |

### 16.5 何时才分库分表？

一般规则：

- 单表行数 > 5000 万～1 亿
- 单库容量接近硬件上限
- QPS/TPS 单实例撑不住

**过早分片**代价很大（运维复杂、开发受限），先优化 SQL/索引/缓存/硬件。

### 📝 笔试题 16-1：用"用户 ID 取模 16 分片"有哪些坑？

- **扩容难**：从 16 → 32 需全量 rehash
- **热点用户**：大 V 所在分片压力大
- **按其他维度查询**：按订单 ID/商品 ID 查必须扫所有分片
- **一致性哈希** + 预留虚节点可缓解扩容；**基因法**可让订单带上用户分片信息

---

## 17. 备份、恢复与运维

### 17.1 逻辑备份

```bash
# 导出整库
mysqldump -u root -p --single-transaction --routines --triggers \
  --master-data=2 dbname > dump.sql

# 导入
mysql -u root -p dbname < dump.sql

# 仅结构 / 仅数据
mysqldump -d ...      # schema
mysqldump -t ...      # data
```

`--single-transaction` 对 InnoDB 一致性备份；对 MyISAM 无效。

### 17.2 物理备份

- **XtraBackup / MariaBackup**：在线热备，适合大库
- **MySQL Enterprise Backup**：官方商业
- **Percona XtraBackup**：开源

```bash
xtrabackup --backup --target-dir=/backup/full
xtrabackup --prepare --target-dir=/backup/full
```

### 17.3 时间点恢复（PITR）

```bash
# 全量 + binlog 回放到某个时间点
mysql < full.sql
mysqlbinlog --start-datetime='...' --stop-datetime='...' \
  mysql-bin.000123 | mysql
```

### 17.4 闪回

- binlog 格式为 `ROW` 时，可用工具（`binlog2sql`、`my2sql`）把 DML 反解，生成"反向 SQL"
- 对付**误 DELETE/UPDATE** 的救命稻草

### 17.5 升级与维护

- 升级前：`mysqlcheck --check-upgrade`
- 灰度切流：先升级从库，观察再主库
- 定期优化：`ANALYZE TABLE`、删除无用索引、归档冷数据

### 📝 笔试题 17-1：`mysqldump` 对 InnoDB 一致性备份关键参数？

`--single-transaction` 启动一个 REPEATABLE READ 事务做快照，期间 DML 不影响备份一致性。**注意**：DDL 仍会破坏一致性；备份时不做 DDL。

---

## 18. 安全与权限

### 18.1 用户与权限

```sql
CREATE USER 'app'@'10.0.%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON orders_db.* TO 'app'@'10.0.%';
REVOKE DELETE ON orders_db.* FROM 'app'@'10.0.%';
FLUSH PRIVILEGES;         -- 一般自动，显式调用保险

SHOW GRANTS FOR 'app'@'10.0.%';
```

### 18.2 最小权限原则

- 应用账号只拿需要的库/表/列权限
- 禁用 `GRANT ALL`
- 分环境隔离（dev/stg/prd 独立账号，最好独立实例）
- DDL 权限仅给 DBA

### 18.3 密码与传输安全

- 8.0 默认认证插件 `caching_sha2_password`，比旧 `mysql_native_password` 安全
- 强制 TLS：`REQUIRE SSL`（或 `X509`）
- 数据库**不暴露公网**；必要时走 bastion/VPN

### 18.4 防注入

**SQL 注入**的根本对策是**参数化查询**（prepared statement），而非字符串转义。

```java
// JDBC
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE email=?");
ps.setString(1, email);
```

```python
# Python (PEP 249)
cursor.execute("SELECT * FROM users WHERE email=%s", (email,))
```

### 18.5 审计

- `general_log`：记录全部 SQL（性能损耗大）
- 企业版 Audit Plugin
- 云 RDS 多带审计开关

### 📝 笔试题 18-1：最小化 SQL 注入风险的三条措施？

1. **参数化查询**（根本）
2. **最小权限**（即使注入也伤害受限）
3. **输入白名单校验** + **错误信息不回显**

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** InnoDB 默认隔离级别？
A. READ UNCOMMITTED  B. READ COMMITTED  C. REPEATABLE READ  D. SERIALIZABLE

<details><summary>答案</summary>C。</details>

**Q2** 下面哪个会造成索引失效？
A. `WHERE a=1 AND b=2`（联合索引 (a,b)）
B. `WHERE DATE(created_at)='2025-01-01'`
C. `WHERE id IN (1,2,3)`
D. `WHERE id BETWEEN 10 AND 20`

<details><summary>答案</summary>B。函数作用在索引列。</details>

**Q3** 下列关于 `COUNT` 的说法**正确**的是？
A. `COUNT(1)` 比 `COUNT(*)` 快
B. `COUNT(col)` 含 NULL 值
C. `COUNT(DISTINCT col)` 不含 NULL
D. `SELECT COUNT(*) FROM t` 不扫任何数据

<details><summary>答案</summary>C。A 现代版本等价；B 不含 NULL；D 仍需扫索引。</details>

**Q4** 关于 B+Tree 索引，错误的是？
A. 叶子节点形成有序链表
B. 非叶节点不存数据
C. 主键索引叶子存整行（InnoDB）
D. 支持 `LIKE '%xx%'` 使用索引

<details><summary>答案</summary>D。前缀通配无法用 B+Tree 索引。</details>

**Q5** 关于事务，错误的是？
A. REPEATABLE READ 下快照读可重复
B. `START TRANSACTION` 开启事务
C. `autocommit=1` 时单条 DML 自动提交
D. `ROLLBACK` 后仍可 `COMMIT`

<details><summary>答案</summary>D。回滚后事务结束。</details>

**Q6** 关于外键，不正确的是？
A. MyISAM 不支持
B. 可以 `ON DELETE CASCADE`
C. 跨库仍可以用外键
D. 高并发写场景通常不建议用

<details><summary>答案</summary>C。外键不能跨库。</details>

**Q7** 下列语句哪个不是 DDL？
A. `CREATE INDEX ...`  B. `ALTER TABLE ...`  C. `TRUNCATE TABLE ...`  D. `DELETE FROM ...`

<details><summary>答案</summary>D。DELETE 是 DML。</details>

**Q8** 联合索引 `(a, b, c)`，以下不能走索引的查询？
A. `WHERE a=1 AND b=2 AND c=3`
B. `WHERE a=1 AND c=3`（索引用到 a）
C. `WHERE b=2 AND c=3`
D. `WHERE a=1 ORDER BY b`

<details><summary>答案</summary>C。违反最左前缀。</details>

### 19.2 判断题

1. `TRUNCATE` 可以回滚。 ❌
2. InnoDB 的二级索引叶子节点存完整行。 ❌（存主键）
3. `SELECT ... FOR UPDATE` 是当前读。 ✅
4. MySQL 8.0 已删除 Query Cache。 ✅
5. `utf8mb4` 才是真正的 UTF-8。 ✅
6. 主键一定不能被修改。 ❌（物理可以，强烈不建议）
7. 事务中的 DDL 会被回滚。 ❌（多数 DDL 隐式提交）
8. 从库 `Seconds_Behind_Master` 等于 0 就一定没延迟。 ❌（有多种干扰因素）

### 19.3 简答题

**Q1** 简述 InnoDB 如何实现"可重复读"。

通过 **MVCC + Undo Log**：每行有隐藏事务 ID，事务首次 SELECT 时生成 Read View，之后该事务内的快照读都按这个 Read View 判断可见性，读到的不再变化，从而"可重复读"。

**Q2** 写一条 SQL：每个用户最近一次登录时间，包含从未登录的用户。

```sql
SELECT u.id, u.username, MAX(l.login_at) AS last_login
FROM users u
LEFT JOIN login_logs l ON l.user_id = u.id
GROUP BY u.id, u.username;
```

**Q3** 大表加索引卡住业务，如何处理？

- 8.0 Online DDL `ALGORITHM=INPLACE, LOCK=NONE`
- 或用 `pt-online-schema-change` / `gh-ost`（无锁或低锁）
- 避开高峰时段
- 主从架构可先从库加，再切主

**Q4** 长事务的危害？

- **锁不释放**：阻塞其他写，易死锁
- **Undo log 无法清理**：版本链长，历史记录膨胀
- **MDL 阻塞 DDL**：引发连环阻塞
- **主从延迟**：从库重放慢
- 监控：`SELECT * FROM information_schema.innodb_trx WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;`

### 19.4 SQL 题

**Q1** 每个部门薪资前三高的员工（经典题）。

```sql
SELECT dept_id, name, salary
FROM (
  SELECT dept_id, name, salary,
         DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
  FROM employees
) t
WHERE rk <= 3;
```

**Q2** 连续登录 7 天的用户（经典题）。

```sql
WITH t AS (
  SELECT user_id, login_date,
         DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY) AS grp
  FROM (SELECT DISTINCT user_id, DATE(login_at) AS login_date FROM login_logs) x
)
SELECT user_id
FROM t
GROUP BY user_id, grp
HAVING COUNT(*) >= 7;
```

**思路**：相邻日期若连续，则 `date - rn` 相同，按 `(user_id, grp)` 分组计数 ≥ 7。

**Q3** 每日 UV / 新增用户 / 留存用户。

```sql
SELECT DATE(login_at) AS d,
       COUNT(DISTINCT user_id) AS uv,
       COUNT(DISTINCT CASE WHEN first_login_date = DATE(login_at) THEN user_id END) AS new_users,
       COUNT(DISTINCT CASE WHEN first_login_date <  DATE(login_at) THEN user_id END) AS retained
FROM login_logs l
JOIN users u ON u.id = l.user_id
WHERE login_at >= CURDATE() - INTERVAL 30 DAY
GROUP BY DATE(login_at)
ORDER BY d;
```

**Q4** 订单表有 id、user_id、amount、created_at，找出"同一用户 1 小时内连续下单 ≥ 3 笔"的记录。

```sql
SELECT *
FROM (
  SELECT o.*,
         COUNT(*) OVER (PARTITION BY user_id
                        ORDER BY created_at
                        RANGE BETWEEN INTERVAL 1 HOUR PRECEDING AND CURRENT ROW) AS cnt_1h
  FROM orders o
) t
WHERE cnt_1h >= 3;
```

> 注：`RANGE INTERVAL` 窗口帧 MySQL 8.0+ 支持。

---

## 📚 学习建议

1. **动手为主**：装一个本地 MySQL 8，造 100 万行数据，测索引/事务/锁的实际行为
2. **读执行计划**：`EXPLAIN` 是 SQL 优化的第一视角
3. **一个 SQL 多种写法**：子查询 / JOIN / 窗口函数，比较性能
4. **看日志**：慢查询日志、binlog、`SHOW ENGINE INNODB STATUS` 是 DBA 的三件套
5. **读官方文档**：`InnoDB` 与 `Optimization` 章节是最权威资料
6. **对照业务**：OLTP、OLAP、报表、时序的需求差异，别把所有表当订单表设计

> 祝你的 SQL 写得又快又稳。
