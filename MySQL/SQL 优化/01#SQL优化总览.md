# **面向 Java 开发的 SQL 优化清单**

## **0）定位与复现（你能做的）**

- 从云厂商控制台/应用 APM 拿到**最慢 SQL 文本 + 绑定参数**与出现频率（Top N）。
- 在测试或影子数据上用同样参数跑 EXPLAIN / EXPLAIN ANALYZE（8.0 可用），关注：
  - 访问类型：ALL（全表扫）/range/ref/eq_ref
  - 关键指标：**rows**（估算行数）、**filtered**（过滤比例）、using filesort / using temporary
  - 回表：Extra 里是否显示 Using index（覆盖）/ Using where（需回表）
- **只改 SQL 或索引**，对比前后执行计划与时延；保留原 SQL 作为回滚。





## **1）语法级“可走索引化”（SARGable）改写**

- **时间过滤改范围**：DATE(create_time)=? →

  create_time >= ? AND create_time < ? + INTERVAL 1 DAY

- **避免列上函数/计算**：LEFT(phone,3)='138' → phone >= '138' AND phone < '139'

- **避免隐式类型转换**：列是 INT 就传 INT，别传 '123' 字符串；否则可能放大扫描。

- **OR 拆解**（两侧各可走索引时）：

```
-- 原：status = 1 OR user_id = ?
SELECT ... FROM t WHERE status = 1
UNION ALL
SELECT ... FROM t WHERE user_id = ? AND status <> 1;
```

- **分页大偏移改“锚点翻页”**（Keyset Pagination）：

```
-- 有条件(user_id) + 排序(created_at,id) 的复合索引
SELECT ... 
FROM orders 
WHERE user_id = ? AND (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

- **尽量覆盖索引**：只查必要列，SELECT * 改为列清单；能被二级索引覆盖就不回表。
- **排序/分组对齐索引**：WHERE 条件 + ORDER BY 的列顺序与方向尽量与索引一致；否则常见 filesort。
- **EXISTS/IN 取舍**：
  - 小集合常量：IN (?, ?, ?) OK。
  - 相关子查询：多半 EXISTS 更稳；或改 JOIN 再去重。
- **反连接**：NOT IN（含 NULL）可能翻车 → 用 NOT EXISTS。





## **2）索引设计（只动必要 DDL）**

> 只做**精确命中这条慢 SQL**的最小索引；避免把写入压垮。

- 连接键双方都建索引，**类型/字符集一致**（避免隐式转换）。
- 复合索引顺序：**等值列在前** → **范围列靠后** → **排序/分组列最后**；尽量覆盖查询列。

```
-- 例：查用户订单，按时间倒序分页
CREATE INDEX idx_orders_u_ct_id ON orders(user_id, created_at DESC, id DESC);
```

- 删除**冗余索引**（被前缀完全覆盖的、从未使用的）。
- JSON 热路径：可建**生成列 + 索引**（可选，业务允许再上）。

```
ALTER TABLE t 
  ADD COLUMN sku_id BIGINT 
    GENERATED ALWAYS AS (JSON_EXTRACT(extra,'$.skuId')) STORED,
  ADD INDEX ix_sku_id(sku_id);
```





## **3）JOIN 与子查询专项**

- **先小后大**：先在驱动表把行数压小（子查询/CTE 内先过滤）。
- **JOIN 前过滤**：把能下推的 WHERE 提前到子查询里，减少回表与临时表。
- **必要时用 STRAIGHT_JOIN / USE INDEX** 强制驱动表或索引（仅当计划反复选错，且你已验证正确性）。

```
SELECT /*+ SET_VAR(optimizer_switch='') */ STRAIGHT_JOIN ...
-- 或：FROM t FORCE INDEX (idx_x) ...
```





## **4）常见慢点“对症改法”（可直接套用）**

- **模糊搜索**：LIKE '%kw%' → 使用倒排（ES）或改为前缀 kw% + 索引；保底用二段法：先前缀匹配筛候选，再二次匹配。
- **计数接口**：COUNT(*) WHERE ... 尽量让条件走窄索引；必要时单做**轻量汇总表**（写触发/异步维护）。
- **去重+排序**：能通过复合索引满足就让引擎用索引顺序；否则考虑先聚合再小表 JOIN。
- **大 IN 列表**（上千）：改**临时表** + JOIN（会话临时表即可）。





## **5）Java 侧能立刻改的（不动数据库参数）**

- **PreparedStatement 全面化**：避免字符串拼接；确保参数类型与列类型一致。

- **批量写入真批量**（MyBatis/JDBC）

  - 连接串：rewriteBatchedStatements=true
  - 代码：addBatch()/executeBatch()；MyBatis 用 ExecutorType.BATCH

  

- **大结果集流式读取**（避免一次性拉爆内存/网络）

```
// DataSource URL 增加 useCursorFetch=true
// 代码层设置 fetchSize（MySQL 8 驱动需二者配合）
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setFetchSize(500);  // 服务端游标
    try (ResultSet rs = ps.executeQuery()) { ... }
}
```

- **分页接口统一成 Keyset 模板**；MyBatis 提供 (lastCreatedAt,lastId) 两个锚点参数。
- **MyBatis 动态 SQL 避坑**：LIKE CONCAT('%', #{kw}, '%') 会走不了索引；把“前缀匹配 + 二次过滤”的两段逻辑写清楚。
- **JPA/Hibernate（如用）**：开 jdbc.batch_size、order_inserts、order_updates；查询尽量用投影接口而非实体全量映射。





## **6）验证闭环（你能做的）**

- 同参对比 EXPLAIN ANALYZE：是否从 ALL → range/ref、rows 是否显著下降、filesort/temporary 是否消失。
- 看应用端：平均耗时/P95 是否下降、数据库连接等待是否下降（线程池阻塞是否缓解）。
- 留好**回滚**：索引名、旧 SQL 文本、验证截图/日志。





# **面试可复述的 60 秒答案（背下来就能用）**

> **问：慢 SQL 如何优化？**

> **答：我分三步走**：

> 1）**定位与证据**：从控制台/APM 抓**慢 SQL + 参数**，用 EXPLAIN/EXPLAIN ANALYZE 找瓶颈点（全表扫/回表/临时表/排序）。

> 2）**只改 SQL 与索引**：把表达式改成可走索引（时间改范围、避免列上函数、OR 拆 UNION、分页改 Keyset）；为该语句补一个**复合覆盖索引**（等值在前、范围其后、排序最后），JOIN 两端键都建索引且类型一致。

> 3）**验证与守护**：对比计划与时延（filesort/temporary 消失、扫描行数下降），在应用侧统一模板化（Keyset 分页、批量写、流式读）避免回退。

> **有例子**：订单分页从 LIMIT 100000,20 改 Keyset，并建 (user_id, created_at DESC, id DESC) 索引，P95 从 1.8s 降到 120ms。





# **典型前后对比示例**

**Before（慢）：**

```
SELECT * 
FROM orders 
WHERE user_id = ? AND DATE(create_time) = ?
ORDER BY create_time DESC
LIMIT 50 OFFSET 50000;
```

**After（快）：**

```
-- 索引
CREATE INDEX idx_orders_u_ct_id ON orders(user_id, create_time DESC, id DESC);

-- SQL
SELECT id, create_time, amount, status
FROM orders
WHERE user_id = ?
  AND create_time >= ? AND create_time < ? + INTERVAL 1 DAY
  AND (create_time, id) < (?, ?)              -- 上一页最后一行的锚点
ORDER BY create_time DESC, id DESC
LIMIT 50;
```



走索引：

避免where 里使用函数（DATE(create_time)=?）

避免列上函数、列上计算

避免隐式类型转换

使用 OR 拆分 sql，让多段查询走各自的索引

最左前缀原则，复合索引尽量把非等值查询的列放后面

连接查询用索引

慎用 LIKE

尽量让去重、分组、排序用上索引

最左前缀原则（索引name age type 查询条件name type，ICP 虽然能够直接在辅助索引树上继续下探，无需回表，但是却不能减少扫描行数）



扫描的少：

锚点翻页（多字段排序的情况下）

```sql
SELECT  ... 
FROM    orders
WHERE   ...                                 -- 你的其它过滤条件
  AND (ctime < :last_ctime
       OR (ctime = :last_ctime AND id < :last_id))
ORDER BY ctime DESC, id DESC
LIMIT 200;
```

大IN改用EXISTS（小心 NULL）（本质还是想让小驱动大）

连接查询时小表驱动大表

派生表优化（where user_id > (select ...)）

子查询优化（不要让结果集的每一行都再执行一次单独的子查询）

ICP、DCP

