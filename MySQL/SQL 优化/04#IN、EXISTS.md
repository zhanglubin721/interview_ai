# **IN ↔ EXISTS 快速指南（含 NULL 要点）**



## **为什么把** **IN**改 **EXISTS**

- 表达“是否存在匹配行”更直接，优化器更易做**半连接**。
- 避免某些场景的不必要 GROUP BY/物化。
- 处理 NOT IN 时能**规避 NULL 大坑**（见下）。







## **基本等价改写模板**

```
-- IN → EXISTS
-- a.col IN (SELECT b.key FROM B b WHERE 条件)
WHERE EXISTS (
  SELECT 1 FROM B b
  WHERE b.key = a.col        -- 相关谓词（等值）
    AND 条件
);

-- NOT IN → NOT EXISTS（推荐）
-- a.col NOT IN (SELECT b.key FROM B b WHERE 条件)
WHERE NOT EXISTS (
  SELECT 1 FROM B b
  WHERE b.key = a.col
    AND 条件
);
```

**复合键（多列）**

```
-- (a.k1, a.k2) IN (SELECT k1, k2 FROM B WHERE 条件)
WHERE EXISTS (
  SELECT 1 FROM B b
  WHERE b.k1 = a.k1 AND b.k2 = a.k2
    AND 条件
);
```

**仅判断存在性时，移除子查询里的 GROUP BY**

```
-- 原：IN (SELECT user_id FROM orders WHERE ct>=? GROUP BY user_id)
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.user_id = u.id AND o.ct >= ?
);
```

**大批量 IN（>1000 常量）→ 临时表/派生表 + EXISTS**

```
-- 将常量批量写入 temp_keys(k)
WHERE EXISTS (SELECT 1 FROM temp_keys t WHERE t.k = a.k);
```





## **NULL 三值逻辑（核心结论）**

- SQL 是三值逻辑：TRUE / FALSE / UNKNOWN。WHERE 中 UNKNOWN 会被**当作不匹配**过滤掉。

- **IN 与 NULL**

  - 列表/子查询里有 NULL：比较结果可能变成 UNKNOWN，**效果等同于“不命中”**（通常无害）。

  

- **NOT IN 与 NULL（大坑）**

  - 只要子查询结果**包含一个 NULL**，整体比较就可能是 UNKNOWN，WHERE 里被过滤 → **可能返回 0 行**，与预期相反。
  - **稳妥写法**：把 NOT IN 改 NOT EXISTS；或确保子查询列 **NOT NULL** / 显式排除 NULL。

  



**安全替代示例**

```
-- 危险：子查询可能产出 NULL
WHERE u.id NOT IN (SELECT o.user_id FROM orders o WHERE o.status='PAID');

-- 安全首选
WHERE NOT EXISTS (
  SELECT 1 FROM orders o
  WHERE o.user_id = u.id AND o.status='PAID'
);

-- 或者（若坚持 NOT IN）显式去 NULL
WHERE u.id NOT IN (
  SELECT o.user_id FROM orders o
  WHERE o.status='PAID' AND o.user_id IS NOT NULL
);
```

**允许 NULL 也视为相等时**（复合键/可空列）

```
WHERE NOT EXISTS (
  SELECT 1 FROM dict d
  WHERE d.k1 <=> a.k1     -- NULL-safe 等号
    AND d.k2 <=> a.k2
);
```







## **索引与写法建议（落地即用）**

- **被驱动表**（写在 EXISTS 子查询里的表）上的**连接键**必须有索引：

  B(key) 或复合索引 (key, 其他过滤列)。

- 谓词保持 **SARGable**（列不上函数/计算；时间用范围；前缀可索引）。

- 复合键匹配就按**多列等值**写清楚（b.k1=a.k1 AND b.k2=a.k2），便于命中复合索引。

- 大量常量改**临时表 + EXISTS/JOIN**，别堆超长 IN (...)。







## **验证要点（EXPLAIN/ANALYZE）**

- IN/EXISTS 在 8.x 通常会被优化为**半连接**；EXPLAIN FORMAT=TREE/JSON 会看到 semi join/firstmatch 等字样。
- NOT EXISTS → 反半连接（anti-join）；避免 NOT IN 的 NULL 陷阱。
- 关注：key 命中、type 为 ref/eq_ref、rows 明显下降、Using join buffer 消失。
- EXPLAIN ANALYZE 看实际 rows/loops，确认没有“外层行数 × 子查询代价”的爆炸。







### **记忆版总结**

- **能 EXISTS 就 EXISTS**（尤其替代 NOT IN）。
- **NOT IN + NULL = 雷区**；改 NOT EXISTS 或显式去 NULL。
- **复合键写多列等值**；**索引在子查询表上**。
- IN 碰到 NULL 多数只是不命中；**NOT IN 碰到 NULL 常常全军覆没**。