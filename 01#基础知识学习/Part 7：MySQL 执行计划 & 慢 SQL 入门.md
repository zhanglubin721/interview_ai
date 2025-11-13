# 慢 SQL 优化

EXPLAIN关键列（会读就能改）

- **type（访问方式，从好到差）**：system/const > eq_ref > ref > range > index > ALL。追求 **ref/range 及以上**，避免 ALL。

  【关键词】type 等级、ALL=全表扫

- **key/possible_keys**：实际用到的索引/可选索引；key_len 说明**用到了几列**。

  【关键词】key_len、多列索引利用度

- **rows & filtered**：预估扫描行数 × 选择率；乘积越小越好。

  【关键词】rows×filtered

- **Extra**：

  - Using index（覆盖索引） ✅

  - Using index condition（ICP，仍会回表）

  - Using where（回表后再过滤）

  - Using temporary / Using filesort（排序/分组未走索引）❌

    【关键词】覆盖索引、ICP、临时表/文件排序



EXPLAIN ANALYZE

（8.0+ 必学）

- **作用**：给出**真实**行数与耗时，定位“估值与真值偏差”。

- **看点**：每个算子 rows/actual time/loops；找**最长支路**与**最大 rows examined**。

- **常见问题**：统计信息失真 → ANALYZE TABLE；索引选择错误 → 重写谓词/调索引顺序。

  【关键词】实际 vs 预估、ANALYZE TABLE、最慢支路



先把规则讲清：EXPLAIN 的 **type** 是“每张表本次访问的方式”，一行代表一个表；从好到差大致是

system/const > eq_ref > ref > range > index > ALL（中间还有 ref_or_null、index_merge 等，这里略）。**越靠左，扫描的行越少、越依赖索引的等值命中**。



下面用一组简单表举例，并展示**如何把 type 往左提**。

```sql
CREATE TABLE user(
  id BIGINT PRIMARY KEY,
  email VARCHAR(128) UNIQUE,
  name  VARCHAR(64),
  INDEX idx_name(name)
);

CREATE TABLE orders(
  id BIGINT PRIMARY KEY,
  user_id BIGINT,
  status TINYINT,
  create_time DATETIME,
  INDEX idx_status(status),
  INDEX idx_ct(create_time),
  INDEX idx_user_ct(user_id, create_time)
);
```



## 各 type 的典型 SQL

**const（单表、唯一等值）**

命中主键/唯一键的等值，优化器在优化阶段就能确定“最多一行”：

```sql
EXPLAIN SELECT * FROM user WHERE id = 123;           -- type = const
EXPLAIN SELECT * FROM user WHERE email = 'a@b.com';  -- type = const
```

**eq_ref（连接时的“唯一等值”）**

被连接的那张表通过主键/唯一键等值被定位到**恰好一行**：

```sql
EXPLAIN
SELECT o.* FROM orders o
JOIN user u ON u.id = o.user_id
WHERE o.id = 100;    -- u 的 type = eq_ref（按唯一键 id 取一行）
```

**ref（等值命中非唯一/联合索引的左前缀）**

等值条件，但落在**非唯一索引**或联合索引的左前缀上，可能返回多行：

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;                 -- type = ref
EXPLAIN SELECT * FROM user   WHERE name = 'Tom';               -- type = ref
EXPLAIN SELECT * FROM orders WHERE user_id = 42;               -- type = ref（走 idx_user_ct 左前缀）
```

**range（范围扫描）**

BETWEEN、>、<、IN（多个离散值）或前缀 LIKE（'abc%'）：

```sql
EXPLAIN SELECT * FROM orders
WHERE create_time >= '2025-11-01' AND create_time < '2025-12-01';  -- type = range

EXPLAIN SELECT * FROM user WHERE name LIKE 'Tom%';                  -- type = range
```

**index（全索引扫描）**

没有过滤条件或条件用不到索引，只能把**整棵索引**从头扫到尾；比 ALL 好在**只读索引页**、可能覆盖：

```sql
EXPLAIN SELECT name FROM user;         -- 只有读 name 时，可能走 idx_name 全索引扫，type = index
```

**ALL（全表扫描）**

既无索引可用又有过滤，或者写了不“可索引化”的条件（函数包裹、前缀 %like）：

```sql
EXPLAIN SELECT * FROM orders WHERE YEAR(create_time) = 2025;  -- type = ALL（函数破坏索引）
EXPLAIN SELECT * FROM user   WHERE name LIKE '%Tom%';         -- type = ALL（前缀通配）
```

> 小提示：看 **rows** 与 filtered 列更能量化代价；同为 range，rows=10 远胜 rows=100000。





## **如何把 type 往左提（可操作清单）**

- **ALL → range/ref**

  - 去掉函数、做“可索引化（SARGable）”改写：

    YEAR(create_time)=2025 → create_time >= '2025-01-01' AND create_time < '2026-01-01'（建 idx_ct）。

  - LIKE '%abc' → 业务允许的话改成 'abc%'；不行就建 **FULLTEXT** 或引入搜索引擎。

  - 给过滤列建索引/联合索引，并确保类型/字符集一致，避免隐式转换。

  

- **index → range**

  - 加上合理的过滤条件或利用**排序索引**：

    ORDER BY create_time 配合 WHERE create_time >= ? 可成为 range，并避免临时表/文件排序（Extra 不再 Using filesort）。

  

- **range → ref**

  - 尽量把范围条件变成**等值命中**（例如先把候选集合落入临时表，再 JOIN 等值）。
  - 把**等值列放在联合索引的最左**：例如常写 WHERE user_id=? AND create_time BETWEEN ... → 建 (user_id, create_time)，则 user_id 等值命中，create_time 范围续扫。

  

- **ref → eq_ref（在 JOIN 中）**

  - 让被连接列具备**唯一性**：把 user.id 保持 PK/UNIQUE；连接条件用等值且无函数/计算。
  - 保证数据类型、字符集、排序规则一致，避免因为隐式转换而放弃唯一索引。

  

- **eq_ref/const 的拿法**

  - **const**：在单表条件里用主键/唯一键等值常量：WHERE id=? / WHERE email=?。
  - **system**：只有一行的小表，极少见，意义不大。

  



## **典型“改一条就见效”的重写**

- **函数包裹列 → 区间**

```
-- 坏
WHERE DATE(create_time) = '2025-11-01'
-- 好
WHERE create_time >= '2025-11-01' AND create_time < '2025-11-02'
```

- **OR 拆分 → UNION ALL（让每支用上索引）**

```
-- 坏：status 有索引但 OR 让选择变差，可能退化
WHERE status = 1 OR status = 2
-- 好：两支都是等值命中，可各自 ref/range
(SELECT ... WHERE status = 1)
UNION ALL
(SELECT ... WHERE status = 2)
```

- **IN 大集合 → 临时表 + 等值 JOIN**

```
-- 坏：IN(大量值) 可能成大范围
WHERE user_id IN ( ...上千个... )
-- 好：把集合落临时表 tmp(u_id PK)，然后
JOIN tmp t ON t.u_id = orders.user_id   -- type = ref/eq_ref（取决于唯一性）
```

- **联合索引顺序**

  把查询里“**等值**→**范围**→**排序/分组**”的列按这个顺序建：

  WHERE user_id=? AND status=? AND create_time BETWEEN ... ORDER BY create_time DESC

  → 索引 (user_id, status, create_time)，通常 type = range，并能用上索引序完成排序。



## 半连接

```sql
SELECT * 
FROM employees e
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.emp_id = e.id
);
```

这条 EXISTS 在 MySQL 8.0 会先被**半连接（semi-join）改写**，然后在多种半连接策略里择优。你的这个等值关联 o.emp_id = e.id，如果 orders(emp_id) 有索引，**最常见会选 FirstMatch**：以 employees 为驱动表，对每个 e.id 用索引在 orders 里做一次等值查找，**命中一条就停止**（不产生重复行），语义与 EXISTS 完全等价。

这时候如果你自己强行写成了

```sql
SELECT DISTINCT e.*
FROM employees e
JOIN orders o ON o.emp_id = e.id;
```

- **只做“存在性过滤”时**：EXISTS/IN 更合适。8.0 会把它们改写成**半连接（semi-join）**，常走 **FirstMatch/LooseScan**——命中一条就停，不产重复行。你若手写 JOIN，为了语义等价还得再加 DISTINCT/GROUP BY 去重，这一步**可能更慢**，也会限制优化器选择更高效的半连接策略。
- **需要右表列 / 右表是 1:1 或 N:1（右表连接键有唯一索引）**：手写 INNER JOIN 与 EXISTS 通常**一样快**（甚至更快一点），因为连接类型会落到 eq_ref/ref，不产生重复，也便于做**覆盖索引**（直接从右表索引里取所需列，少回表）。
- **手写** **JOIN** **但关系是 1:N**（右表无唯一约束）：不加去重会**多返回行（语义错）**；加了 DISTINCT/GROUP BY 后，优化器就**不能用“命中即停”的 FirstMatch**，可能退化到“连接再去重”，这时**通常比** **EXISTS** **慢**。



## 派生表合并

```sql
SELECT dept_id, AVG(salary)
FROM (SELECT * FROM employees WHERE salary > 5000) t
GROUP BY dept_id;
```

mysql 会自动把 sql 改写成

```sql
SELECT dept_id, AVG(salary)
FROM employees WHERE salary > 5000
GROUP BY dept_id;
```

- **内联（merge/inline）**：优化器发现派生表不需要真正物化，就把它的查询语句直接合并到外层查询中。
- 这样减少了临时表开销，提高性能。

**内部逻辑**

- MySQL 5.6 之前：派生表一定会物化成临时表 → 慢。
- MySQL 5.7+ 引入 **derived_merge** 优化：能内联就尽量内联。
- 限制：如果派生表包含 LIMIT、GROUP BY、DISTINCT 等语义，可能必须物化。



## 去相关化

```sql
SELECT * FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e.dept_id
);
```

把相关子查询（外层每一行都需要执行一遍子查询）改写成 join

```sql
SELECT e.* 
FROM employees e
JOIN (
    SELECT dept_id, AVG(salary) AS avg_sal 
    FROM employees GROUP BY dept_id
) x ON e.dept_id = x.dept_id
WHERE e.salary > x.avg_sal;
```



# 优化案例

## 1) 列表分页 + 排序很慢（ORDER BY ... LIMIT offset,n）

**场景**：订单列表 WHERE seller_id=? ORDER BY gmt_create DESC LIMIT 20 OFFSET 100000。

**症状**：扫描/回表+filesort，offset 越大越慢。

**根因**：无法利用有序索引直接定位页首；offset 需要丢弃前 N 行。

**优化**：改“位移分页”为“锚点/Seek 分页”，并建立复合索引覆盖过滤与排序。

```sql
-- 索引
CREATE INDEX ix_seller_ctime ON t_order(seller_id, gmt_create DESC, id);

-- 传统写法（慢）
SELECT cols... FROM t_order
WHERE seller_id=? 
ORDER BY gmt_create DESC, id DESC
LIMIT 20 OFFSET 100000;

-- 优化写法（Seek分页）
SELECT cols... FROM t_order
WHERE seller_id=? 
  AND (gmt_create,id) < (?, ?)   -- 上一页最后一条的锚点
ORDER BY gmt_create DESC, id DESC
LIMIT 20;
```

要点：过滤列放前，范围列放最后；用覆盖索引避免回表；Seek 分页是大厂通用套路。参考 ORDER BY/LIMIT 优化与 Seek 分页原理。 





## **2) 排序+LIMIT 仍 filesort（范围+排序）**

**场景**：WHERE status=1 AND gmt_create BETWEEN ... ORDER BY gmt_create DESC LIMIT 50。

**症状**：执行计划 Using where; Using filesort。

**根因**：没有合适的复合索引；或先扫二级索引再随机回表 I/O 多。

**优化 A（首选）**：复合覆盖索引 (status, gmt_create DESC, id)；顺序与谓词/排序一致。

**优化 B（延迟关联 / Late Join）**：先用小而窄的覆盖索引取到主键，再回表拿全列。

```sql
-- A 覆盖索引
CREATE INDEX ix_status_ctime ON t_order(status, gmt_create DESC, id);

-- B 延迟关联
SELECT o.* FROM (
  SELECT id FROM t_order
  WHERE status=1 AND gmt_create BETWEEN ? AND ?
  ORDER BY gmt_create DESC, id DESC
  LIMIT 50
) k JOIN t_order o USING(id);
```

要点：配合 MRR/ICP 进一步减少随机 I/O 与回表。 





## **3)** OR连接导致走 Index Merge 或全表扫

**场景**：WHERE user_id=? OR type=?。

**症状**：index_merge 合并中间结果巨大、CPU/I/O 峰值高。

**优化**：改写为 UNION ALL，分别命中各自索引，再合并（必要时再 DISTINCT/去重）。

```sql
-- 索引
CREATE INDEX ix_user ON t_log(user_id);
CREATE INDEX ix_type ON t_log(type);

-- 优化写法
SELECT ... FROM t_log WHERE user_id=?
UNION ALL
SELECT ... FROM t_log WHERE type=?;
```

UNION ALL 常比 OR 可预测且稳定。 





## **4) 在列上使用函数，索引失效（非 SARGable）**

**场景**：WHERE DATE(create_time)=?，或 WHERE UPPER(name)=?。

**症状**：type=ALL 扫描多、延迟高。

**优化**：改写为可走“范围/等值”的谓词；必要时用“函数索引（8.0+）”。

```sql
-- 好的写法（范围）
SELECT ... FROM t WHERE create_time >= '2025-11-01' AND create_time < '2025-12-01';

-- 或函数索引（示例）
CREATE INDEX ix_upper_name ON t ((UPPER(name)));
```

要点：让“列在左、常量/变量在右”，避免对列做函数/计算。 





## **5) 模糊搜索** LIKE '%kw%'慢

**场景**：商品检索 name LIKE '%iPhone%'。

**症状**：前导 % 使 B-Tree 无法利用有序前缀；退化为全表/全索引扫描。

**优化**：改成全文索引 FULLTEXT（或外置检索如 ES）；能前缀匹配的用 LIKE 'kw%' 并建索引。

```sql
ALTER TABLE product ADD FULLTEXT ft_name(name);
SELECT ... FROM product WHERE MATCH(name) AGAINST('+iphone' IN BOOLEAN MODE);
```





## **6) 超长** IN ( ... )列表导致解析/执行慢

**场景**：应用层传 1w～5w 个 id：WHERE id IN (....)。

**症状**：计划膨胀、扫描重复、CPU 飙升。

**优化**：把 id 写入临时/中间表，建索引后 JOIN；或批量插入 MEMORY 临时表后 JOIN。

```sql
CREATE TEMPORARY TABLE tmp_ids (id BIGINT PRIMARY KEY);
-- 批量插入 tmp_ids 后：
SELECT t.* 
FROM main t JOIN tmp_ids x USING(id);
```

实测大列表转临时表 + JOIN 常显著更快、更省内存。 





## **7) 相关/非相关子查询很慢**

**场景**：WHERE user_id IN (SELECT ...)、EXISTS (SELECT ...)。

**症状**：重复执行子查询、或物化代价大。

**优化**：依赖 MySQL 的半连接（Semi-Join）/物化；或显式改写成 JOIN + 去重。

```sql
-- 改写示例
SELECT /*+ JOIN_FIXED_ORDER() */ DISTINCT u.*
FROM user u 
JOIN orders o ON o.user_id=u.id
WHERE o.status=1;
```

理解半连接策略（FirstMatch、DuplicateWeedout、LooseScan、Materialization）有助于判断是否需要手动改写。 





## 8) “每组 Top1/最新一条”查询慢（MAX/GROUP BY）

**场景**：每个店铺取“最新一笔订单”。

**优化 A（松散索引扫描/Loose Index Scan，单表场景）**：为 (shop_id, gmt_create DESC) 建索引，直接走松散扫描。

**优化 B（派生表 + 回表）**：先 GROUP BY shop_id 求 MAX(gmt_create)，再回表定位整行。

```sql
-- B：常见稳定写法
WITH m AS (
  SELECT shop_id, MAX(gmt_create) AS mx
  FROM t_order GROUP BY shop_id
)
SELECT o.* FROM t_order o
JOIN m ON m.shop_id=o.shop_id AND m.mx=o.gmt_create;
```

了解何时能触发 Loose Index Scan 与其限制条件。 

```sql
-- 第一步：各组最大时间
SELECT shop_id, MAX(gmt_create) AS mx_ctime
FROM t_order
GROUP BY shop_id;

-- 最终查询（处理同一时间的并列，取 id 最大）
SELECT o.*
FROM t_order o
JOIN (
  SELECT shop_id, MAX(gmt_create) AS mx_ctime
  FROM t_order
  GROUP BY shop_id
) x
  ON o.shop_id = x.shop_id AND o.gmt_create = x.mx_ctime
LEFT JOIN t_order o2
  ON o2.shop_id = o.shop_id
 AND o2.gmt_create = o.gmt_create
 AND o2.id > o.id
WHERE o2.id IS NULL;   -- 把并列里“不是最大的”排掉
```

```sql
-- NOT EXISTS 写法
SELECT o.*
FROM t_order o
WHERE NOT EXISTS (
  SELECT 1
  FROM t_order x
  WHERE x.shop_id = o.shop_id
    AND (x.gmt_create > o.gmt_create
         OR (x.gmt_create = o.gmt_create AND x.id > o.id))
);
```

```sql
SELECT o.*
FROM t_order o
LEFT JOIN t_order x
  ON x.shop_id = o.shop_id
 AND (x.gmt_create > o.gmt_create
      OR (x.gmt_create = o.gmt_create AND x.id > o.id))
WHERE x.shop_id IS NULL;
```



## 9) 统计计数慢（COUNT(\*)/ COUNT(DISTINCT ...)**）**

**场景**：大表做全表 COUNT(*) 或频繁 COUNT(DISTINCT user_id)。

**症状**：InnoDB 需遍历索引并做可见性判断；并发下更慢。

**优化**：

- COUNT(*) 让优化器走“最小二级索引”更快；

- 高频实时指标 → 预聚合/明细落地 + 异步刷新；

- 高频去重统计 → 预计算/周期性物化（必要时借助数据仓库/流式聚合）。

  



## **10) 随机推荐** ORDER BY RAND()慢

**场景**：从 1000 万行中随机取 10 条。

**症状**：RAND() 为每行生成随机数再排序，代价巨大。

**优化**：避免对全量排序：

- 基于主键范围的“随机跳跃”再 LIMIT；

- 预先维护“随机池/采样表”；

- RAND()*max_id 命中后补偿重试。

  



## **11) 连接顺序/驱动表选择不佳**

**场景**：大表与多个小表连接，优化器偶尔选错驱动表，导致爆炸式中间结果。

**优化**：

- 补充统计信息、正确索引；

- 必要时用 STRAIGHT_JOIN / JOIN_FIXED_ORDER / JOIN_ORDER 提示限定连接顺序；

- 验证后再固化。

  



## **12) 额外的引擎层优化开关（让“回表更少、I/O更顺序”）**

- **ICP（Index Condition Pushdown）**：把能在索引层判断的条件下推，减少不必要回表。

- **MRR（Multi-Range Read）**：先收集主键排序再顺序回表，降低随机 I/O。

  理解它们在 EXPLAIN 的体现（Using index condition / Using MRR）。 





# **排查与落地清单（实战化）**

1. **定位**：打开慢日志 → 抓 Top N；配合 EXPLAIN ANALYZE/optimizer_trace 看真实代价。
2. **三件套**：**谓词可索引化（SARGable）**、**覆盖**、**顺序一致（where/order/group 与索引列顺序一致）**。
3. **分页**：全部改“Seek 分页”；超大页直接禁止。 
4. **子查询**：尽量走半连接或改 JOIN；OR → UNION ALL。 
5. **搜索**：%like% 上全文/外置检索；前缀匹配用索引。 
6. **统计**：实时指标预聚合/物化，COUNT(*) 利用最小二级索引或离线化。 