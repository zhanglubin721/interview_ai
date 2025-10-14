# **总体执行链（一眼看懂）**

动态代理生成的 mapper 接口实现类内部的具体执行流程

```
Mapper 接口方法
   ↓（JDK 动态代理：MapperProxy）
SqlSession（DefaultSqlSession 或 Spring 的 SqlSessionTemplate）
   ↓
Executor（Simple/Reuse/Batch）  ← 二级缓存、插件拦截点
   ↓
MappedStatement（SQL 元数据）
   ↓
SqlSource → BoundSql（最终 SQL + 参数映射表）
   ↓
StatementHandler（prepare/parameterize/batch/update/query）
   ↓
ParameterHandler（把 Java 参数 → JDBC 占位符）
   ↓
JDBC PreparedStatement.execute()
   ↓
ResultSetHandler（ResultSet → Java 对象）
   ↑
返回 Mapper 方法的返回类型
```



# **1) Mapper XML 与 Mapper 接口的绑定**

**绑定规则**

- mapper.xml 的 namespace 必须等于 **Mapper 接口的全限定名**。

- XML 中 <select|insert|update|delete id="方法名"> 的 id 必须与**接口方法名**一致。

- 返回类型与参数类型由：

  - XML 上的 parameterType/resultType/resultMap；
  - 以及接口方法的**签名**共同决定（最终以 MappedStatement 为准）。

  

**运行时怎么找到 SQL？**

- MapperProxy 拦截接口调用 → 根据 namespace.id 定位 MappedStatement。
- MapperMethod 根据方法签名决定执行 SqlSession.selectOne/selectList/update/...。



**最小示例**

```xml
// 1) 接口
public interface OrderMapper {
    OrderDO selectById(Long id);
    List<OrderDO> query(@Param("uid") Long userId,
                        @Param("from") LocalDateTime from,
                        @Param("to")   LocalDateTime to,
                        @Param("stat") Integer status);
}
<!-- 2) XML（resources/mappers/OrderMapper.xml） -->
<mapper namespace="com.acme.dao.OrderMapper">

  <resultMap id="OrderMap" type="com.acme.dao.OrderDO">
    <id property="id" column="id"/>
    <result property="userId" column="user_id"/>
    <result property="amount" column="amount"/>
    <result property="createTime" column="create_time"/>
  </resultMap>

  <select id="selectById" resultMap="OrderMap">
    SELECT id,user_id,amount,create_time FROM t_order WHERE id = #{id}
  </select>

  <select id="query" resultMap="OrderMap">
    SELECT id,user_id,amount,create_time
    FROM t_order
    <where>
      <if test="uid != null">AND user_id = #{uid}</if>
      <if test="from != null">AND create_time &gt;= #{from}</if>
      <if test="to != null">AND create_time &lt; #{to}</if>
      <if test="stat != null">AND status = #{stat}</if>
    </where>
    ORDER BY create_time DESC, id DESC
    LIMIT 100
  </select>

</mapper>
```



# **2) 动态 SQL（SqlNode → SqlSource → BoundSql）**

**内部机制（简述）**

- XML 解析成一棵 **SqlNode** 树（IfSqlNode、WhereSqlNode、ForEachSqlNode 等）。
- 调用时用入参构造 DynamicContext，遍历 SqlNode 生成**最终 SQL 字符串**。
- 动态内容完成后，解析 #{} 参数占位 → 产生 **BoundSql**（? + 参数映射列表）。

**常用标签（要点与坑）**

- <if> 条件拼接；<where> 自动处理前导 AND/OR 与空 WHERE。
- <trim prefix="SET" suffixOverrides=",">：更新语句防尾逗号。
- <foreach>：批量 IN (...) 或批量 VALUES/UPDATE。
- <choose><when/><otherwise/>：互斥条件。
- <bind>：在 OGNL 上下文中预定义变量（如 %kw%）。
- <include refid="...">：SQL 片段复用。
- **永远使用 #{}**（预编译占位）而不是 **${}**（字符串拼接，易注入且破坏参数类型）。

**foreach 示例**

```xml
<select id="listByIds" resultMap="OrderMap">
  SELECT ... FROM t_order
  WHERE id IN
  <foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
  </foreach>
</select>
```

# **3) 参数绑定（ParamNameResolver → ParameterHandler → TypeHandler）**

**参数命名与取值**

- 无 @Param：位置命名 param1/param2/...，并行提供 arg0/arg1/...。
- 有 @Param("name")：按你声明的名字进入上下文。
- 复杂对象：按 OGNL 访问属性（如 #{user.id}）。
- Map 入参：按 key 取值（#{key}）。

**类型到 JDBC 的映射**

- ParameterHandler 根据 BoundSql.parameterMappings 顺序，把每个参数交给 **TypeHandler**：
  - IntegerTypeHandler → PreparedStatement#setInt
  - LongTypeHandler → setLong
  - StringTypeHandler → setString
  - LocalDateTimeTypeHandler/DateTypeHandler → setTimestamp
  - BigDecimalTypeHandler → setBigDecimal
- 可自定义 @MappedTypes/@MappedJdbcTypes 的 TypeHandler<T>。

**#{} vs ${}（必须牢记）**

- \#{}：预编译 + 安全 + 保留索引使用（SARGable）。
- ${}：文本替换（**仅用于** 安全的标识符如表名、列名或 ORDER BY 方向，值必须白名单校验）。

**null 与 jdbcType**

- 参数可能为 null 时，建议显式 jdbcType 防止驱动不确定：

  \#{status, jdbcType=INTEGER}。



# **4) 结果映射（ResultSetHandler）**

- resultType：简单映射（列名 = 属性名或下划线转驼峰）。
- resultMap：复杂映射，含：
  - <association> 一对一（嵌套对象）
  - <collection> 一对多（List/Set）
  - <discriminator> 结果多态
- 驼峰：mapUnderscoreToCamelCase=true（Spring Boot 配置项）
- 嵌套查询慎用 Lazy-Load，容易触发 N+1；更推荐 **单 SQL JOIN + resultMap**。





# **5) 数据源与事务（MyBatis 内部到 Spring 统一）**

**MyBatis 原生**

- DataSource：PooledDataSource（自带简易连接池）、UnpooledDataSource、JNDI。
- TransactionFactory：JdbcTransactionFactory（自己管理）或 ManagedTransactionFactory（容器托管）。
- 实战里基本交给 Spring/Spring Boot：数据源（HikariCP）与事务（@Transactional）。



**Executor 类型**

- SIMPLE：每次新建 Statement。
- REUSE：复用 Statement。
- BATCH：批量提交（配合 MyBatis 的 ExecutorType.BATCH 与驱动的 rewriteBatchedStatements=true）。





# **6) 与 Spring Boot 的整合（mybatis-spring-boot-starter）**

**自动配置做了什么**

- 创建 SqlSessionFactory（由 DataSource 注入，默认 Hikari）。
- 创建 SqlSessionTemplate（线程安全、参与 Spring 事务）。
- 扫描 Mapper：@Mapper 或 @MapperScan("com.acme.dao")。
- 读取配置：mybatis.configuration.*、mapper-locations、type-aliases-package 等。

**典型配置**

```yaml
spring:
  datasource:
    url: jdbc:mysql://.../db?useSSL=false&characterEncoding=utf8&serverTimezone=UTC&rewriteBatchedStatements=true&useCursorFetch=true
    username: xxx
    password: yyy

mybatis:
  mapper-locations: classpath:mappers/*.xml
  type-aliases-package: com.acme.dao
  configuration:
    map-underscore-to-camel-case: true
    default-executor-type: REUSE
    cache-enabled: false
```

**启动类**

```java
@SpringBootApplication
@MapperScan("com.acme.dao")
public class App { public static void main(String[] args){ SpringApplication.run(App.class,args); } }
```

**事务**

```java
@Service
public class OrderService {
  @Resource private OrderMapper orderMapper;

  @Transactional
  public void createOrder(...) {
    orderMapper.insert(...);
    // 发生 RuntimeException/unchecked 时 Spring 触发回滚
  }
}
```





# **7) 缓存与插件（简述）**

- **一级缓存**：SqlSession 级别，默认开启；同一会话、相同语句/参数会命中。
- **二级缓存**：namespace 级别，需 <cache/> 开启，生产要谨慎（与写入一致性/过期策略有关）。
- **插件 Interceptor**：可拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler，实现审计、SQL 日志、分表路由等。





# **8) 常见坑 & 实用建议**

1. **${} 注入与失去预编译**：只在白名单场景使用（列名/排序方向），业务值一律 #{}。
2. **动态 SQL 空 WHERE**：用 <where>/<trim> 防止出现 WHERE AND ... 或无 WHERE 的全表更新。
3. **参数命名**：尽量 @Param 显式命名，避免 param1/arg0 易错。
4. **JOIN 列类型一致**：两端类型/字符集不一致会触发列侧转换、索引失效。
5. **分页**：改 Keyset 分页模板（(create_time,id) < (?,?)），XML 里封装成可复用片段。
6. **批量写**：ExecutorType.BATCH + JDBC URL rewriteBatchedStatements=true + addBatch/executeBatch。
7. **大结果集**：useCursorFetch=true + fetchSize（驱动要求二者配合），避免一次性拉爆内存。
8. **时间类型**：DATETIME ↔ LocalDateTime；慎混用 TIMESTAMP。
9. **N+1**：尽量用单 SQL + resultMap 的 association/collection，或一次查主键再批量二次查询。





# **9) 端到端工作示例（可运行骨架）**

```xml
// domain
@Data
public class OrderDO {
  private Long id;
  private Long userId;
  private BigDecimal amount;
  private LocalDateTime createTime;
}

// mapper
public interface OrderMapper {
  OrderDO selectById(@Param("id") Long id);
  int insert(OrderDO o);
  List<OrderDO> pageByUser(@Param("uid") Long userId,
                           @Param("afterCt") LocalDateTime afterCt,
                           @Param("afterId") Long afterId,
                           @Param("limit") int limit);
}
<!-- resources/mappers/OrderMapper.xml -->
<mapper namespace="com.acme.dao.OrderMapper">

  <resultMap id="Base" type="com.acme.dao.OrderDO">
    <id column="id" property="id"/>
    <result column="user_id" property="userId"/>
    <result column="amount" property="amount"/>
    <result column="create_time" property="createTime"/>
  </resultMap>

  <select id="selectById" resultMap="Base">
    SELECT id,user_id,amount,create_time FROM t_order WHERE id = #{id}
  </select>

  <insert id="insert" parameterType="com.acme.dao.OrderDO" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO t_order(user_id,amount,create_time)
    VALUES(#{userId},#{amount},#{createTime})
  </insert>

  <!-- keyset 分页 -->
  <select id="pageByUser" resultMap="Base">
    SELECT id,user_id,amount,create_time
    FROM t_order
    WHERE user_id = #{uid}
      <if test="afterCt != null and afterId != null">
        AND (create_time, id) &lt; (#{afterCt}, #{afterId})
      </if>
    ORDER BY create_time DESC, id DESC
    LIMIT #{limit}
  </select>

</mapper>
```

好的，我们就把这三件事讲到“源码级可还原”的程度：



# MappedStatement**到底长什么样

MyBatis 把“一个 <select|insert|update|delete> 节点 + 它的上下文”固化为一个 MappedStatement。核心字段（按你最关心的）：

```
MappedStatement
├─ id               : String             // 唯一ID = namespace + "." + sqlId
├─ sqlSource        : SqlSource          // 产出 BoundSql 的工厂（动态 or 静态）
├─ commandType      : SqlCommandType     // SELECT/INSERT/UPDATE/DELETE
├─ statementType    : StatementType      // PREPARED | STATEMENT | CALLABLE
├─ resultMaps       : List<ResultMap>    // 结果映射(可多个)
├─ parameterMap     : ParameterMap       // 参数映射（很少直接用）
├─ timeout          : Integer            // 超时
├─ fetchSize        : Integer
├─ resultSetType    : ResultSetType
├─ databaseId       : String             // 多数据源差异化用
├─ lang             : LanguageDriver     // 默认 XMLLanguageDriver
├─ keyGenerator     : KeyGenerator       // Jdbc3KeyGenerator/SelectKey/NoKey
├─ keyProperties    : String[]           // 自增/回写属性
├─ keyColumns       : String[]           // 对应列名
├─ cache            : Cache              // 二级缓存（可空）
├─ useCache         : boolean
├─ flushCacheRequired: boolean
└─ resource         : String             // 定义来源（XML文件名:行号）
```

- **sqlSource** 才是真正关键：它在“执行时”生成 BoundSql（最终 SQL + 参数映射）。
- **resultMaps** 决定 ResultSet 怎么转成 DO。
- **statementType=PREPARED** 是默认，意味着走 PreparedStatement。

你可以把 MappedStatement 理解为：**可执行 SQL 的“定型描述 + 执行策略”**。



# **SQL 解析成** SqlNode**树到底是什么**

MyBatis 解析 <mapper> 的 <select> 等标签时，会把其中的“动态内容”编译成一棵 **SqlNode 抽象语法树**。常见节点：

- MixedSqlNode：复合节点（根）
- StaticTextSqlNode：纯文本 SQL 片段（不含 ${})
- TextSqlNode：可能含 ${} 的文本
- IfSqlNode / ChooseSqlNode / TrimSqlNode / WhereSqlNode / SetSqlNode
- ForEachSqlNode：循环展开 IN (...)、批量 VALUES 等

**判定是否“动态 SQL”的规则**：出现 <if|choose|foreach|trim|where|set> **或** 文本里含 ${}，就走 **DynamicSqlSource**；否则走 **RawSqlSource**（静态）。

**例子（从 XML 到树，再到最终 SQL）**

```
<select id="query" resultMap="OrderMap">
  SELECT id,user_id,amount,create_time
  FROM t_order
  <where>
    <if test="uid != null">AND user_id = #{uid}</if>
    <if test="from != null">AND create_time <![CDATA[>=]]> #{from}</if>
    <if test="to != null">AND create_time <![CDATA[<]]> #{to}</if>
  </where>
  <if test="orderBy != null">
    ORDER BY ${orderBy}   <!-- 注意：${} 直拼，需白名单 -->
  </if>
  LIMIT #{limit}
</select>
```

**解析后的树（示意）：**

```
MixedSqlNode
├─ StaticTextSqlNode("SELECT id,user_id,amount,create_time FROM t_order")
├─ WhereSqlNode
│  └─ MixedSqlNode
│     ├─ IfSqlNode(test="uid != null")
│     │  └─ StaticTextSqlNode("AND user_id = #{uid}")
│     ├─ IfSqlNode(test="from != null")
│     │  └─ StaticTextSqlNode("AND create_time >= #{from}")
│     └─ IfSqlNode(test="to != null")
│        └─ StaticTextSqlNode("AND create_time < #{to}")
├─ IfSqlNode(test="orderBy != null")
│  └─ TextSqlNode("ORDER BY ${orderBy}")   // 动态文本（含 ${}）
└─ StaticTextSqlNode("LIMIT #{limit}")
```

**执行时（传入参数 Map/对象）**：

1. DynamicContext 初始化，把 @Param 命名的参数塞入上下文。
2. 深度遍历 SqlNode：
   - IfSqlNode 通过 **OGNL** 求值（uid != null 等）决定是否拼接子节点文本；
   - WhereSqlNode 会自动处理前导 AND 并补上 WHERE；
   - TextSqlNode 若含 ${}，会把 ${orderBy} 直接**字符串替换**为值（危险，必须白名单）；
   - 所有 #{} 此时仍保留原样。
3. 得到“**动态拼接后的 SQL 字符串**”（仍含 #{}）。
4. 走 SqlSourceBuilder.parse(...)：把 #{...} 统一替换为 ?，并**生成参数映射列表**。
5. 产出 BoundSql：
   - sql: 例如 SELECT ... WHERE user_id = ? AND create_time >= ? ORDER BY create_time DESC LIMIT ?
   - parameterMappings: 每个 ? 对应一个 ParameterMapping（属性名、javaType、jdbcType、typeHandler……）
   - additionalParameters: foreach 的 item/index、_parameter 等运行期临时变量

**注意**：无动态标签且无 ${} 的 SQL，会在**解析阶段**就把 #{} 编译成 StaticSqlSource，执行时不再做第 2～4 步，开销更小。



# **“预编译”在干嘛？为什么** **#{}**能防注入

**概念分层**（非常关键）：

- **MyBatis 层**：把 #{} 变成 ?，并为每个 ? 准备好 ParameterMapping（类型、TypeHandler）。
- **JDBC 驱动层**：PreparedStatement 持有 SQL 模板（带 ?）。调用 setInt/setString/setTimestamp... 把**值**绑定到参数槽位。
- **数据库层**（以 MySQL 为例）：
  - **Server-side PreparedStatement（真预编译）**：
    1. COM_STMT_PREPARE：服务端解析 SQL 语法 → 生成内部语法树并分配 stmt-id；
    2. 多次 COM_STMT_EXECUTE：只送**二进制参数**，无需再拼接 SQL 字符串；
    3. 由引擎做优化/执行（是否复用计划由引擎决定）。
  - **Client-side emulation（驱动侧模拟）**：驱动把参数**安全转义**后拼成最终 SQL 再一次性发送。**仍然安全**，但缺少 prepare/execute 的协议分离与潜在的解析复用。

> MySQL Connector/J 默认并**不强制**服务器端预编译；但无论“真预编译”还是“驱动模拟”，**只要你用 PreparedStatement/#{}，参数都不会被当成 SQL 语法的一部分**。





**#{}**防注入的本质

- \#{} → ? → **参数和值分离**：

  攻击载荷（比如 "1 OR 1=1"）被当作**纯字符串/纯数值**传入参数槽，不参与 SQL 解析，数据库把它看作 "1 OR 1=1" 这个**值**，而不是语法。

- TypeHandler + JDBC setter 决定“以什么类型”传值：

  - 数值就走 setInt/setLong；
  - 字符串会自动加引号或使用二进制协议传输；
  - 时间走 setTimestamp。

- 反例：${} 是**文本拼接**，把值直接插入 SQL 字符串 → **参与解析** → 注入风险。

**小演示**

```
-- Mapper XML
SELECT * FROM user WHERE name = #{name};

-- 攻击者传入 name = "a' OR '1'='1"
-- 最终发送给DB：
-- 预编译SQL：SELECT * FROM user WHERE name = ?
-- 参数：[ "a' OR '1'='1" ]  ← 这是一个“含引号的字符串”，不会被当作 OR 语句
```

而如果写成：

```
SELECT * FROM user WHERE name = '${name}';
-- 这会变成：
SELECT * FROM user WHERE name = 'a' OR '1'='1';
-- 直接被数据库解析成永真条件
```



**预编译的副作用/边界**

- **执行计划复用**：是否真正复用由数据库决定（不同参数可能触发不同计划，例如范围变化很大的情况）。
- **类型匹配影响索引**：PreparedStatement 能保持类型一致性，通常更利于索引命中（避免隐式转换）。
- **驱动模拟也安全**：即便没启用 server-side prepare，驱动的**参数化 API**也会做严密转义/编码，避免把值注入到语法层。



# @DS

**结论版**

- @DS 就是一面“路由旗帜”：AOP 在方法**进入前**把数据源 key 放进一个 ThreadLocal 栈（进入时 push，退出时 pop）。

- 第一次需要连接时，AbstractRoutingDataSource.determineCurrentLookupKey() 从 ThreadLocal 取 key，返回对应物理数据源（取不到就用默认）。

- **是否能在同一线程内切换库，取决于有没有已绑定的事务连接**：

  

  - **没有事务**：每次执行 SQL 都会单独拿/还连接，此时你可以在不同方法上用不同 @DS，**可切换**。
  - **有事务（@Transactional 已开启）**：事务开始时**第一次**拿到的连接被绑定到线程，**后续切 @DS 不生效**（仍用已绑定连接），除非打破/嵌套事务边界。

  

- **跨库“一个事务里同时操作多个库”的 ACID**：用动态数据源 + DataSourceTransactionManager 做不到；需要 XA/JTA（Atomikos/Narayana）或分布式事务框架（如 Seata）。否则只是多个**本地**事务/自动提交，**不具备原子性**。



## **细节与常见场景**

### **1) AOP 顺序与注解位置**

- 一般把 @DS 标在 **Service 方法**（和 @Transactional 同层），动态数据源的切面优先级会设置得**高于**事务切面，让“定路由”发生在拿连接之前。
- 也可标在 Mapper 上，但要确保切面顺序正确；否则事务先拿了默认库连接，再切库就**来不及**。



### **2) “同一线程多库”如何写才生效**

- **无事务**：

```
@DS("db1") public void opDb1() { mapperA.doX(); }    // 用 db1
@DS("db2") public void opDb2() { mapperB.doY(); }    // 用 db2

public void biz() { opDb1(); opDb2(); }              // 可切换（各自拿/还连接）
```

- **有事务**（外层加了 @Transactional）：

```
@Transactional
public void biz() {
  opDb1();           // 第一次取连接，绑定到线程（例如 db1）
  opDb2();           // 即使 @DS("db2")，也仍然用已绑定的 db1 连接 → 不切换
}
```

- 想在同一调用链里切库，有两种“非原子”的办法：
  - 用不同**事务传播**开启**新事务**：

```
@DS("db1") @Transactional
public void opDb1(){ ... }

@DS("db2") @Transactional(propagation = REQUIRES_NEW)
public void opDb2(){ ... } // 新事务，单独拿 db2 连接
```

- 注意：这是两个独立本地事务，**不是**跨库原子提交。
- 或者让其中一段**不加入事务**（propagation = NOT_SUPPORTED / 默认无事务），各自拿独立连接。





### **3) 真正的跨库原子事务**

- 需要 **JTA/XA**（JtaTransactionManager + XA 数据源）或 **Seata**（AT/TCC/SAGA）一类的协调器。
- 动态数据源只是**路由**，不提供两阶段提交；DataSourceTransactionManager 也只会管理**一个**数据源的本地事务。



### **4) 嵌套/回退与线程边界**

- 动态数据源实现一般用**栈式 ThreadLocal**，支持嵌套调用时“内层覆盖、退出还原”。
- ThreadLocal **不跨线程**：异步任务/线程池中需要在子线程里重新设置 @DS（或使用可传递的上下文方案）；否则就用默认源。



### **5) 读写分离的常见坑**

- 写后立刻读若路由到从库，可能读到**旧数据**（主从延迟）。解决：写事务内或写后的一段时间内强制走主库（@DS("master")），或用“读你的写”策略。





# MyBatis 动态代理逻辑

## **一、@DS 与 @Transactional 在 AOP 里的“排序原理”**

**谁来排序？**

Spring 用的是 AnnotationAwareAspectJAutoProxyCreator（简称 AutoProxyCreator）。它在 Bean 初始化完成后：

1. 为该 Bean **收集所有可用的 Advisor**（切点 + 通知），包括：

   - 事务的 BeanFactoryTransactionAttributeSourceAdvisor（内部包着 TransactionInterceptor）
   - 动态数据源框架提供的 DynamicDataSourceAnnotationAdvisor（内部包着设置/清理 ThreadLocal 的拦截器）
   - 你写的其它 @AspectJ 切面

   

2. 用 AnnotationAwareOrderComparator **按优先级排序这些 Advisor**：

   - 谁的 order 值**越小**（更接近 Ordered.HIGHEST_PRECEDENCE），谁越先执行（包在最外层）。
   - 事务 Advisor 的顺序来自 @EnableTransactionManagement(order=...)（默认 **LOWEST_PRECEDENCE**，即很靠后）。
   - 动态数据源的 Advisor（@DS）在主流实现里会把 order 设成**很高优先级**（数值很小），**确保它先于事务**运行——这样在事务去拿连接之前，数据源 key 已经塞进 ThreadLocal 了。

> 直白点：**默认配置下，@DS 的切面先执行，事务切面后执行**。



## **二、最终“代理长什么样”？一次成型，不是一层层套娃**

**是不是“先用 @DS 生成一个实现类，再被 @Transactional 再包一层”？**

不是。**常规情况下只创建**一个 Spring AOP 代理（JDK 或 CGLIB），**把所有 Advisor 串成一条拦截链**，链路顺序就是上面排好序的优先级。

- 代理创建入口：AbstractAutoProxyCreator#wrapIfNecessary(...)
- 选 Advisor：AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean(...)
- 排序：AnnotationAwareOrderComparator.sort(advisors)
- 织入：ProxyFactory 把这些 Advisor 依序加入，产出**单个**代理对象

**调用栈示意（@DS 优先、事务其后）：**

```
[Spring AOP Proxy]
  ├─ DS MethodInterceptor（设置 ThreadLocal 路由键）
  │     try {
  │        └─ TransactionInterceptor（开启/加入事务 → 第一次取连接）
  │             └─ 目标方法（你的 Service/Mapper 方法）
  │           （提交/回滚）
  │     } finally {
  │        清理 ThreadLocal
  │     }
```

**例外：Mapper 的情况**

MyBatis 的 Mapper 本身就是 **JDK 动态代理**（$ProxyXX）。

如果你把 @DS/@Transactional 标在 **Mapper 接口** 上，Spring 还会在 Mapper 这个 **JDK 代理外面**再包一层 **Spring AOP 代理**（“代理包代理”是允许的）。

因此最佳实践是：**把 @DS / @Transactional 标在 Service 层**，让 Spring 只对 Service 创建一次 AOP 代理，链路更清晰、开销更低，也能正确控制事务边界。





## **三、为什么必须“@DS 在外、事务在内”**

- 事务切面真正用到连接的时机是：DataSourceUtils.getConnection(DataSource)。
- 你的 DataSource 往往是 AbstractRoutingDataSource 的子类（动态路由数据源）。它在 determineCurrentLookupKey() 里**读取 ThreadLocal 的数据源 key**。
- **如果事务先运行**并拿到了连接，再设置 @DS 就**太晚了**（此次调用已绑定默认库）。
- 所以要保证 **@DS 切面在事务切面之前执行**，才能影响“首次取连接”的路由。





## **四、代码级“长相”参考（极简伪码）**

**DS Advisor 的拦截器**（等价逻辑）：

```java
class DynamicDsInterceptor implements MethodInterceptor, Ordered {
  int getOrder() { return /* 很小的值，优先级高 */; }
  Object invoke(MethodInvocation inv) throws Throwable {
    String key = resolveDsKey(inv.getMethod(), inv.getThis().getClass());
    DataSourceContextHolder.push(key); // ThreadLocal
    try {
      return inv.proceed();
    } finally {
      DataSourceContextHolder.pop();
    }
  }
}
```

**事务拦截器**（Spring 自带，简化）：

```java
class TransactionInterceptor implements MethodInterceptor, Ordered {
  int getOrder() { return /* 来自 @EnableTransactionManagement，通常最大值 */; }
  Object invoke(MethodInvocation inv) throws Throwable {
    TransactionInfo tx = createTxIfNecessary(...); // 里头第一次取连接 → 触发路由
    try {
      Object ret = inv.proceed();
      commitOrResume(tx);
      return ret;
    } catch (Throwable ex) {
      rollbackOnException(tx, ex);
      throw ex;
    } finally {
      cleanup(tx);
    }
  }
}
```





## **五、两个常见问题澄清**

1. **它是一次生成好链，还是“先生成一个再包一层”？**

   → **一次生成好**。AutoProxyCreator 会把**所有可用的**切面找齐、排好序、一次性织入**同一个代理**。

   只有在目标对象本身就不是 Spring AOP 产物（比如 MyBatis Mapper 的 JDK 代理），并且你又想对它做切面时，才会出现“代理包代理”的情况。

2. **能不能在同一线程内多次切换库？**

   → **无事务**时，每次方法都会独立取/还连接，按各自的 @DS 路由。

   → **有事务**时，**第一次取连接**那一刻就锁定了库；后续 @DS 不会生效，除非开启新事务（REQUIRES_NEW）或不用事务。**跨库原子事务**要靠 XA/JTA/Seata，非本问范畴。





## **六、怎么验证你项目里实际的“排序与落库”**

- 打印 Advisor 顺序：开启 org.springframework.aop 的 debug，能看到为某 Bean 组装的 Advisor 列表及顺序。
- 验证路由是否生效：在一个小的 MyBatis 插件或自定义拦截器里打印 executor.getTransaction().getConnection().getMetaData().getURL()，直观看到最终用的 JDBC URL。
- 若发现 @DS 不生效，检查：
  1. 注解位置是否在 **事务边界外层**（推荐 Service 方法）
  2. 动态数据源 Advisor 的 order 是否 **小于** 事务 Advisor
  3. 是否已经在调用链更早处（比如另一个切面或 @Transactional 的父方法）拿过连接

这样，你就能把“为什么 @DS 要先于事务执行、代理是一次性组装的拦截链”讲到源码级别而不跑题。