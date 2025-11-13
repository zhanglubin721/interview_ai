### #5.2.1

#### Snowflake

常见的 Snowflake 设计把 64 位 long 拆成 3 段（总计 63 位；最高 1 位保持为 0，保证正数）：

```
[63]           [62 .................................. 22] [21 .... 12] [11 .......... 0]
 sign(0)              timestamp(41 bits, 毫秒)            workerId(10)       sequence(12)
```

- **timestamp(41bits)**：从某个**自定义纪元（epoch）开始到现在的毫秒数**。41 位能表示的范围是 0 .. 2^41-1 毫秒，约 **69.7 年**。

  例：如果把 epoch 设为 2020-01-01 00:00:00 UTC，那能用到大概 **2089 年**。

- **workerId(10bits)**：机器/实例编号，范围 0..1023（很多实现会再拆成 5 位机房 + 5 位机器）。

- **sequence(12bits)**：**同一毫秒内**的自增序号，范围 0..4095，也就是**每毫秒最多 4096 个** ID/实例。

```java
// Maven: cn.hutool:hutool-all
import cn.hutool.core.lang.Snowflake;
import org.springframework.context.annotation.Bean;
import org.springframework.beans.factory.annotation.Value;

@Configuration
public class IdConfig {
  @Bean
  public Snowflake snowflake(
      @Value("${id.worker:1}") long workerId,
      @Value("${id.dc:1}") long dcId,
      @Value("${id.clockRollbackToleranceMs:5000}") long toleranceMs // 允许回拨 5s
  ) {
    // isUseSystemClock=true 使用 Hutool 的 SystemClock，降低 System.currentTimeMillis() 开销
    return new Snowflake(null, workerId, dcId, true, toleranceMs);
  }
}
// 使用：snowflake.nextId();
```



### #5.4.1

#### VARCHAR

VARCHAR(100) 表示**最多存 100 个字符（character）**，不是字节；每一行只占**实际内容的字节数 + 1~2 个字节的长度前缀**，**不会预先占满 100**。它不是“会自己扩容”的意思，而是**天生可变长**：行里放多少字符就占多少空间（再加上那 1~2 字节前缀）。





### #5.9.1

#### 二级索引事务可见性判断

**没有把 trx_id 存到“二级索引记录里”**。InnoDB 直到 8.4 LTS（2025）仍然明确：**二级索引记录不含隐藏系统列**（没有 DB_TRX_ID/DB_ROLL_PTR），它们也不是原地更新的；一致性读需要时仍要回到聚簇索引做可见性判断。 



但你看到的说法并非空穴来风，它混淆了一个**页级别**的优化：

二级索引**页头**维护了一个 PAGE_MAX_TRX_ID（该页上记录可能被“最近哪个事务”修改的上界）。在**覆盖索引**场景里，如果这个页的 PAGE_MAX_TRX_ID 对你的 Read View 来说**不算“太新”**，且记录未被 delete-mark，那么引擎可以**直接用索引里的列返回**，从而**避免对每条记录都回表**；一旦发现页“太新”或记录被标删，仍然需要回表到聚簇索引读取 DB_TRX_ID/undo 做 MVCC 校验。 



官方博客与代码也佐证了这一点：

- **不在二级索引的“每条记录”上跟踪 max_trx_id，只在“整页”上跟踪**，所以有时会“误判太新”而保守地回表。 
- 源码里的 lock_sec_rec_cons_read_sees() 注释写得很直白：二级索引页能提供的信息很有限，“判 false 时**必须**去聚簇索引再查一次”。 



另外还有两个现实中的“坑”，也解释了为啥你仍常见回表：

- **长事务/写入活跃**时，很多二级索引页的 PAGE_MAX_TRX_ID 会被视为“过新”，覆盖索引就**不能直接用**。官方手册也专门提醒：这种情况下覆盖索引技术会失效，需要查聚簇索引。 
- **变更缓冲（change buffer）等路径**可能使页头的 PAGE_MAX_TRX_ID 不总是精确上界，实现上倾向**保守**处理（不是每次都更新到最新），从而增加“不得不回表”的概率。 



一句话判断

- 说“二级索引里有最新的 trx id，因而减少回表”：**表述不严谨**。正确说法是：**二级索引“页头”有** **PAGE_MAX_TRX_ID****，有助于在覆盖索引时“有时”免回表**；并**不是**把 trx_id 存进每条二级索引记录。 



给你的实务结论

- 设计覆盖索引依然有价值，但在 RR/RC 下**不能指望完全不回表**；是否能免回表，取决于该页是否“够老”且记录未标删。 
- 读老快照或写入很频繁时，**免回表命中率会下降**；这是引擎保守性换一致性的代价。 



### RC vs RR 的核心差异（聚焦锁行为）

- **RR（可重复读）**：为避免幻读，InnoDB 对“范围访问（包括范围 UPDATE/锁定读）”使用 **next-key 锁（记录锁 + gap 锁）**，锁粒度大、冲突多、等待长，**更容易**出现锁等待/死锁。
- **RC（读已提交）**：**禁用了搜索/扫描时的大多数 gap 锁**（但唯一/外键检查仍可能用 gap）。因此**更少阻塞**，并发更高，但允许**幻读/不可重复读**。



### 审计字段

#### 方案 A：Spring Data JPA Auditing（最省心）

**特点**：自动填充 + 禁止客户端覆盖。

**怎么做**

1. 开启审计：

```java
@EnableJpaAuditing(auditorAwareRef = "auditorAware")
@Configuration
class AuditConfig {
  @Bean AuditorAware<Long> auditorAware() {
    return () -> Optional.of(SecurityContext.getUserId()); // 自己实现
  }
}
```

1. 实体上标注：

```java
@Entity @EntityListeners(AuditingEntityListener.class)
class Order {
  @CreatedDate @Column(nullable=false, updatable=false)
  private Instant createdAt;

  @CreatedBy @Column(nullable=false, updatable=false)
  private Long createdBy;

  @LastModifiedDate @Column(nullable=false)
  private Instant updatedAt;

  @LastModifiedBy @Column(nullable=false)
  private Long updatedBy;
}
```

- @Created* + updatable=false：JPA **不会在 UPDATE 里带这两列**，客户端即使传了也覆盖不了。
- AuditorAware 从登录态取当前用户，确保 **由服务器端写入**。

> 可选：用 Hibernate 的 @CreationTimestamp/@UpdateTimestamp 自动打时间戳；或配合 @PrePersist/@PreUpdate 回调强制写值。



#### 方案 B：MyBatis-Plus 自动填充（MP 项目首选）

**特点**：不改 SQL 手写处，统一填充。

1. 标注字段：

```java
@Data
class Order {
  @TableField(fill = FieldFill.INSERT)          private Instant createdAt;
  @TableField(fill = FieldFill.INSERT)          private Long createdBy;
  @TableField(fill = FieldFill.INSERT_UPDATE)   private Instant updatedAt;
  @TableField(fill = FieldFill.INSERT_UPDATE)   private Long updatedBy;
}
```

1. 全局填充器：

```java
@Component
public class AuditMetaObjectHandler implements MetaObjectHandler {
  public void insertFill(MetaObject meta) {
    this.strictInsertFill(meta, "createdAt", Instant::now, Instant.class);
    this.strictInsertFill(meta, "createdBy", () -> User.id(), Long.class);
    this.strictInsertFill(meta, "updatedAt", Instant::now, Instant.class);
    this.strictInsertFill(meta, "updatedBy", () -> User.id(), Long.class);
  }
  public void updateFill(MetaObject meta) {
    this.strictUpdateFill(meta, "updatedAt", Instant::now, Instant.class);
    this.strictUpdateFill(meta, "updatedBy", () -> User.id(), Long.class);
  }
}
```

- strict*Fill 会在你没显式赋值时**强制覆盖**；结合**接口层 DTO 不暴露这些字段**（见“通用加固”）即可杜绝客户端传值生效。





### 崩溃恢复

**Undo Log（回滚日志）**

InnoDB 在每次改动行时都会写入对应的 *前镜像* 或 *逻辑逆操作*（分为 insert-undo / update-undo）。它有两大作用：① **回滚**未提交事务（撤销到修改前）；② **MVCC 一致性读**——通过 DB_TRX_ID/DB_ROLL_PTR 串起版本链，让快照读能看到“事务开始时”的旧版本。事务提交后，等到没有快照再依赖这些旧版本，**purge 线程**会清理掉相应 undo 版本。



**Redo Log（重做日志）**

Redo 记录的是“**对某个页面的物理变更**”（更准确叫“生理日志”：带页号的逻辑操作），先写入内存 redo buffer，按 **WAL** 规则在脏页落盘前先刷到磁盘（redo 文件循环写，LSN 单调递增）。**崩溃恢复时**按 LSN 从 checkpoint 之后把已提交事务的页修改 **重放** 到数据页，修复“脏页未落盘”的缺口，保证 **持久性**。



**Binlog（归档/复制日志）**

Binlog 是 MySQL Server 层的**逻辑变更日志**（Row/Statement/Mixed，生产常用 Row）。它不参与单机页修复，主要用于 **主从复制** 与 **PITR 恢复**（全备 + binlog 回放到某时点/GTID）。是否“真正落盘”受 sync_binlog 影响。



**三者如何协同保证“崩溃后一致且可恢复”**（两阶段提交的要点）

MySQL 通过 **InnoDB ↔️ Binlog 的两阶段提交（2PC）** 把“引擎内提交”和“写 binlog”捆绑为“要么都成功，要么都不算”。简化时序如下（以常见顺序为例）：

1. 事务执行：写 **undo**（供回滚/MVCC），修改缓冲页，同时生成 **redo 记录**。
2. **Prepare**：InnoDB 写入“prepare 记录”到 **redo** 并刷盘（fsync）。此时事务在引擎内是“可提交”的中间态。
3. Server 层写 **binlog** 并刷盘（受 sync_binlog 控制）。
4. **Commit**：InnoDB 写“commit 记录”到 **redo** 并（按 innodb_flush_log_at_trx_commit）刷盘；事务结束。



**崩溃恢复判定**（启动时）：

- 只看到 **未写入 prepare** 的事务 → 当作未开始，直接忽略或用 **undo** 回滚残留。

- 看到 **prepare**，但**没有持久化的 binlog** → 视为未提交，**用 undo 回滚**。

- 看到 **prepare** 且对应 **binlog 已持久化** → 视为已提交，**应用 redo（完成提交）**。

- 已有 **commit** 记录 → 直接按 **redo** 重放到最新 LSN。

  这样就实现了 **“binlog 与引擎提交的一致性”**：要么两边都生效，要么恢复时回滚。



**几个实务要点（影响“掉电是否丢事务”）**

- innodb_flush_log_at_trx_commit=1 + sync_binlog=1 → 事务提交点两边都 **fsync**，最强持久性；否则掉电可能丢最近事务或出现“主库看起来提交但未落盘”的窗口。
- Redo 只保证**页级修复**，**binlog**才支撑**复制/PITR**；**undo**则支撑**回滚与 MVCC**。三者分工不同，缺一不可。
- 只做“读快照”的查询依赖 **undo 版本链**；恢复阶段也会利用 undo 回滚未提交的修改，随后 **purge** 清理历史版本。



**Undo 让你回到过去（回滚/MVCC），Redo 把过去做到现在（页重放/持久性），Binlog 让你从现在走向任意时点（复制/PITR）**；通过 **2PC** 把 binlog 与 redo 的提交原子化，系统才能在崩溃后既“页一致”，又“逻辑一致”。



- **执行路径（典型 2PC）**

  1. **写 Undo**（形成旧版本，供回滚/MVCC），并把页修改对应的 **Redo** 写入缓冲；
  2. **Prepare**：InnoDB 记一条 *prepare* 到 **Redo** 并**落盘**（保证“已准备好提交”的状态可持久恢复）；
  3. **写 Binlog 并落盘**（含 XID；受 sync_binlog 影响，组提交会把同批次一起 fsync）；
  4. **Commit**：InnoDB 记 *commit* 到 **Redo**（是否立刻 fsync 受 innodb_flush_log_at_trx_commit 影响），释放锁并向客户端返回成功。

  

- **崩溃恢复时的判定**

  - 只有普通修改、**没看到 prepare** → 当作没开始/未准备，按需要用 **Undo 回滚**残留。
  - **有 prepare，但没有持久化的 binlog** → 视为**未提交**，用 **Undo 回滚**。
  - **有 prepare，且对应 binlog 已持久化** → 视为**已提交**：即便来不及写 *commit* 记录，恢复时也会做 **commit-in-recovery**（把事务完成），并用 **Redo** 把脏页补齐。
  - **已有 commit 记录** → 直接按 **Redo** 从 checkpoint 之后重放即可。

  

- **“提交点”如何理解**

  - 从**崩溃一致性语义**看：**第 3 步（binlog 持久化）**是决定“这个事务算不算提交”的分界；
  - 从**客户端**看：只有在**第 4 步完成**后才会收到 COMMIT 成功的返回。
  - 想要“掉电也不丢最近提交”的**最强持久性**，把 innodb_flush_log_at_trx_commit=1、sync_binlog=1 打开；否则掉电可能丢失极近的事务（语义仍一致，但**未必持久**）。

一句话收束：你的流程描述正确——**先让 prepare 可恢复，再让 binlog 持久化，最后写 commit**；恢复时以“**prepare+binlog** 是否同时存在”来决定**回滚还是补交**，从而实现既页级一致（Redo），又逻辑一致（Binlog）的崩溃恢复。