### #2.1.1

#### 代码究竟是什么样的

例子

```java
public class Demo {
    public static void main(String[] args) {
        boolean cond = args.length > 0;   // 只是让它不是编译期常量
        if (cond) {
            System.out.println("A");
        } else {
            System.out.println("B");
        }
    }
}
```

- 你运行 javac Demo.java 得到 Demo.class。
- .class 里是**字节码指令**和**常量池**（方法/字段符号、字符串常量、类型描述符等）。
- Java 的字节码是**基于栈的指令集**（不是寄存器），指令通过“操作数栈”出入栈完成运算和调用。

用 javap -c -v Demo.class（示意）会看到类似下面的主方法片段（简化）：

```
0:  aload_0                      // 加载 String[] args 到操作数栈
1:  arraylength                  // 求长度
2:  ifle     15                  // <=0 跳到 15（即 cond=false 分支）
5:  getstatic  #2 <java/lang/System.out:Ljava/io/PrintStream;>
8:  ldc       #3 "A"             // 常量池里的字符串 A
10: invokevirtual #4 PrintStream.println:(Ljava/lang/String;)V
13: goto      23
15: getstatic #2 System.out
18: ldc       #5 "B"
20: invokevirtual #4 PrintStream.println:(Ljava/lang/String;)V
23: return
```

1. 你写源码 Demo.java。
2. javac 编译成 Demo.class（字节码 + 常量池）。
3. java 启动 JVM 进程，初始化核心类（含 System.out）。
4. 类加载器加载 Demo；字节码验证/准备/解析/初始化。
5. 解释器开始跑 main，创建栈帧，按字节码操作**局部变量表/操作数栈**。
6. 遇到 if* 指令做条件跳转。
7. 调 println：把 PrintStream 和字符串对象当实参，构造新栈帧调用。
8. 方法变热：JIT 收集轮廓信息，触发 C1/C2 编译，生成机器码放入 Code Cache。
9. 后续直接执行机器码（带各类优化），CPU 跑指令、OS 调度、syscall 写 stdout。
10. 你在终端看到“A”或“B”。



#### 字节码、解释器、JIT、机器码

HotSpot 一开始确实用**解释器**按字节码跑；当某段代码被判定为“热点”后，**JIT**（分层编译）把它编成机器码，以后再进入这段逻辑时就**直接走机器码路径**，从而**不再经过解释器那一层的开销**。



1. **分层编译（Tiered Compilation）不是“只有两档”**

- 启动：解释执行（Tier 0）+ 收集 profile（调用次数、分支概率、接收者类型等）。
- 变热：先用 **C1** 快速编译（Tier 1/2/3，带/不带 profiling），让你尽快摆脱解释器开销，同时继续积累更准确的运行期数据。
- 更热：再用 **C2** 做高强度优化（Tier 4），例如内联、去虚拟化、标量替换、逃逸分析、锁消除/粗化、循环优化/向量化、内联缓存（IC）等。
- 结果：**多数热路径会在机器码里跑**；冷代码仍保持解释执行，减少编译/代码缓存开销。



2. **长循环会“中途换引擎”（OSR：On-Stack Replacement）**

- 若你在解释器里跑进了一个很长的 for/while，回边计数变热时，JIT 会生成一个**OSR 入口**的机器码版本。
- JVM 会把当前解释器栈帧**改造成**编译栈帧，**在循环中途切到机器码**继续跑，这样热点循环无需等到方法返回再受益。



3. **并不是“一经编译就永远机器码”——有“去优化（Deopt）”**

- C2 常做**带假设**的激进优化（例如“这里几乎总是某种接收者类型”）。一旦假设失效（来了新类型/新分支），就触发**uncommon trap**：
  - 把执行现场**退回解释器**，按新事实**重新收集 profile / 触发再编译**。
- 所以“机器码路径”与“解释器路径”之间是**可往返**的，保证正确性的同时尽量榨干性能。



4. **“跳过解释器的收益”不仅是少了一层，还来自大优化**

- 真正的加速主要来自：**内联 → 去虚拟化 → 消除分配/锁 → 常量折叠/死码删除 → 循环优化/向量化 → intrinsic（如** **System.arraycopy****/****Math****）**。
- 同时配合**内联缓存（IC/PIC）**把虚调用变成“带类型检查的直呼”，极大降低分派成本。

```
第一次/少数次调用
   ┌───────────┐
   │ 解释器     │  ← 收集profile（调用/回边计数、分支概率、类型）
   └─────┬─────┘
         │ 热度达阈值
         ▼
   ┌───────────┐
   │ C1 快速编译 │  ← 更快进入机器码 + 持续收集更准profile
   └─────┬─────┘
         │ 更热/稳定
         ▼
   ┌───────────┐
   │ C2 高优化  │  ← 激进优化（内联/去虚/标量替换/锁消除…）
   └─────┬─────┘
         │  假设失效（新类型/新分支）
         ▼
   ┌───────────┐
   │ Deopt 回退 │  → 回到解释器，更新profile，可能重编译
   └───────────┘
```

#### OSR

OSR（On-Stack Replacement，**栈上替换**）= 让**正在运行到一半**的方法，**在循环回边处**直接从“解释执行”**热切换**到“已编译的机器码”继续跑（有时也会从低优化级别切到高优化级别）。它解决了“方法只调用一次但里面有一个很长的循环，永远热不起来”的问题。

为什么需要 OSR：仅靠“方法入口计数”触发 JIT 有缺陷：

```java
void work() {
  for (long i = 0; i < 5_000_000_000L; i++) { /* 重活 */ }
}
// 只被调用 1 次，但循环巨大
```

这个方法入口只+1，按“入口阈值”它并不“热”；没有 OSR 的话就会**一直用解释器**慢慢跑完整个循环。

OSR 用**回边计数（back-edge count）盯着循环：当循环迭代很多次，触发阈值，就专门编译一个“从循环中间开始”的版本**，并把执行权切换过去。

```
解释器执行……
   ┌───── 回边（bci=k） ─────┐
   │  back-edge++           │
   │  达阈值→触发OSR编译      │
   └────────┬───────────────┘
            │（编译异步完成）
再次到回边 →  │  已就绪？
            ▼
         是 → 复制状态（locals/stack/monitors）
              跳转到 nmethod 的 OSR entry
              ↓
           机器码继续跑该循环
```

#### TLAB

- **TLAB**：JVM 在**年轻代 Eden**里给**每个 Java 线程**切一小块**私有连续空间**（一个“缓冲区”）。该线程在这块里独享分配对象，**不与别的线程竞争**。
- **bump-the-pointer**：在这块私有空间中维护两个指针：top（已用到哪）和 end（边界）。新对象的分配就是：
  1. 计算对象所需字节数（含对象头、对齐填充）。
  2. 检查 top + size <= end。
  3. 成立则 obj = top; top += size; —— 完整一步 **加法 + 边界判断**。
- 因为**不需要加锁/原子操作**（本线程独享）且**内存已预清零**（后面详述），JIT 能把 new 内联成几条机器指令，**分配速度≈C 的栈上分配**。

### #2.2.1

#### Mark Word：对象的“状态场”

一块随场景复用的 64bit 区域，主要装这些东西（互斥复用）：

1. **无锁状态（unlocked）**

   - 放**GC 年龄（age）**等标志位；
   - 如果**调用过** **System.identityHashCode**，会把**身份哈希**编码进 Mark Word；
   - 老版本（JDK 15 以前）还有**偏向锁（biased）**相关位：偏向 epoch、线程 id 等（JDK 15 起移除偏向锁）。

   

2. **轻量级/栈锁（thin/stack-locked）**

   - 为了做“轻量级同步”，**Mark Word 被临时改写成指向“锁记录（lock record）”的指针**；
   - 原先的 Mark（含可能的 hash/age 等）会被拷贝到“锁记录”的 **displaced header** 里，解锁时写回。

   

3. **重量级锁（inflated/monitor）**

   - 当需要阻塞/等待（或与身份哈希第一次生成发生冲突等）时，锁**膨胀**成 ObjectMonitor；
   - **Mark Word 存的是指向** **ObjectMonitor** **的指针**。这时哈希/状态等信息由监视器结构托管。

> 小结：Mark Word 是 HotSpot 在**同步、哈希、GC 年龄**等多场景下复用的元数据位域。

> 性能含义：在**已加轻量锁的对象上首次算 identityHashCode**，为了安置哈希值，往往会**触发锁膨胀**，后续同步更贵——这是并发热点里常见的“踩坑点”。

#### Klass 指针：对象的“类型/布局/调度身份证”

指向 HotSpot 的 **Klass 元数据**结构（类的运行时描述）。JVM 通过它知道：

- **这是哪个类/是否数组**、如果是数组**元素类型**是什么；
- **对象实例大小**、**字段偏移与类型**、**对齐规则**（JVM 才能在堆上“知道从哪到哪是对象体”）；
- **OopMap/GC 元信息**：哪些偏移上是“引用”（GC 扫描、精确写屏障/读屏障都靠它）；
- **虚方法分派表（vtable）和接口表（itable）**：进行**虚调用/接口调用**的类型分派；
- **常量池、类加载器、修饰位、镜像** **java.lang.Class** 等反射/链接信息；
- **原型 Mark（prototype header）**：新对象初始化时可直接拷贝的“默认头”。

> 小结：klass 指针把**“这个对象是谁/长什么样/该怎么被 GC 和调用”**全部串起来。

> 没有它，JVM既不知道对象占多大、也不知道哪些字段是引用、更没法做虚方法调用。

#### identityHashCode

**语义**：返回**对象身份**的哈希值（与是否重写 hashCode() 无关），**终身稳定**，但**不保证唯一**（可能碰撞）。

**生成时机**：**懒计算**——第一次调用 System.identityHashCode(obj) 或调用 Object.hashCode()（且该类未重写）时才生成；随后把值**写进对象头的 Mark Word 指定位**（无锁态）或写到关联的监视器结构里（重量锁态）。

**为什么不能用“对象地址”当哈希**：HotSpot 的对象会**移动**（复制式/压缩式 GC），地址不稳定。解决：**算一次、存起来**，以后直接读这个值。

**热点实现思路（简化）**：

- 取**JVM 级随机种子**、**（当时的）对象地址/线程噪声**等做一次**混合/扰动**（例如异或与位移、加乘常数等），得到一个分布较好的整数（通常有效位 2531 位）；
- 把这个整数**写入 Mark Word 的哈希位段**；若此时对象处于**轻量级锁**，为安置哈希往往会触发**锁膨胀**（膨胀到 ObjectMonitor 后就有地方放了）。

> 注意：**具体混合公式会随版本/平台有差异**，JVM 也保留调整自由；你应只依赖**“稳定、可碰撞”**这两个语义特征，而不是某个固定公式。

**几个结论**：

- 同一 JVM 进程内：**同一对象**的 identityHashCode **稳定不变**；**不同对象**可能**碰撞**。
- 不同进程/不同启动：由于**随机种子不同**，整体分布会变（有利于安全/散列质量）。
- **性能坑**：在**已加轻量锁**的对象上**首次**计算 identityHashCode，可能导致**锁膨胀**，后续该锁更重。热路径慎用；需要“身份散列”时可**自管一个字段**在构造期写入。

### #2.4.1

#### synchronized

```java
//同步块
synchronized (obj) { body }

//编译后
monitorenter   // 获取 obj 的监视器
... body ...
monitorexit    // 释放 obj 的监视器

//编译器会强制插入 try/finally 确保异常也能执行 monitorexit，否则会“丢锁”：
aload obj
monitorenter
try {
   ... body ...
} finally {
   aload obj
   monitorexit
}
```

自适应自旋

- 目的：**减少阻塞切换**。短暂持有/即将释放的锁，等待方“先转几圈”，可能就拿到了。
- 热点策略（简化）：基于**历史在该监视器上的自旋成功率**、**CPU 核数**、**拥有者是否在运行队列**等启发式决定自旋时长；自旋失败再 park 阻塞。
- 这层策略能显著降低用户/内核态切换开销，但不保证公平性。

> 小结：**无锁→轻锁→（必要时）膨胀**，中途配合**自旋→阻塞**的两段式等待。轻锁用 CAS 和“锁记录”避免系统调用；重锁用 ObjectMonitor 管理队列与 wait/notify。

##### **一、轻量级锁（thin/栈锁）阶段**

**持有者释放（monitorexit 快路径）**

1. 若是重入，弹出一层**锁记录（lock record）**直接返回。
2. 非重入：把对象头中的 **Mark Word** 从“指向我这条锁记录”的指针，**CAS 回写**成锁记录里保存的 **displaced header**（也就是恢复到无锁头）。
   - 成功 ⇒ 释放完成（无内核参与、无唤醒动作）。
   - 失败 ⇒ 说明竞争很激烈或已经被膨胀，走慢路径/膨胀路径。

**竞争者获取（仍为轻锁）**

- 其他线程在进入 synchronized 时会：
  1. 在自己栈上压入一条 lock record，复制目标对象当前 Mark；
  2. **自旋 + CAS** 试图把对象头改成“指向我的 lock record”。
- **当持有者刚刚释放**时，对象头从“指向别人的 lock record”变回“无锁头”，处在自旋圈里的线程会立刻观察到**状态变化**并尝试 CAS 抢占；**第一个 CAS 成功者获锁，其余继续自旋或放弃进入阻塞路径（预算耗尽）**。
- 轻锁阶段**没有显式唤醒**：释放者只做了一个“恢复对象头”的 CAS，**自旋者靠轮询观察头部变化**来竞争。

> 若在轻锁阶段 CAS 冲突过多、或出现 wait()/notify()、或轻锁与 identityHashCode 产生冲突，**会触发膨胀**，转入重量级锁语义。



##### 二、重量级锁（ObjectMonitor）阶段

ObjectMonitor 里有这些关键域：owner（当前持有者）、recursions、EntryList（就绪竞争队列）、cxq（新来的竞争者 LIFO 栈）、以及一个**后继提示** succ（可选，用于减少“插队/车队抖动”）。

**持有者释放（ObjectMonitor::exit）**

1. 若还有重入计数，recursions-- 返回。
2. 否则将 owner = null（释放），形成**释放屏障（release）**以保证临界区写入对后继可见。
3. 如果 EntryList/cxq 中有等待者：
   - 从 EntryList（若空则把 cxq 扫入）选一个**候选者** s；
   - 记录 succ = s（一个“让路给后继”的提示）；
   - **unpark(s)**：把 s 从内核阻塞态唤醒到就绪态。

> 注意：**不是严格的“直接移交（handoff）”**。释放者只是把 owner 置空并唤醒一个等待者；**实际拿到锁的线程仍然要“竞争式获取”**（见下），因此 HotSpot 的监视器**不保证公平**，允许一定程度的**barging（插队）**。

**竞争者获取（两种来源）**

A) **仍在自旋的线程**（短暂忙等）

- 它们会循环检查 owner == null，并使用 CAS 试图把 owner 改成自己：

```
while (spin_budget > 0) {
    if (owner == null && CAS(owner, null, me)) break; // 成功获锁
    pause(); // 短暂停顿，降低总线压力
}
若预算耗尽 ⇒ 入队（cxq/EntryList）并 park
```

- **释放瞬间**，这些自旋者与被唤醒的阻塞者会**一起“抢”**，谁先 CAS 成功谁就成为新 owner。

B) **被唤醒的阻塞线程**（park→unpark）

从**就绪竞争队列** EntryList 里选一个作为候选（若空则把**新来的栈** cxq 倒到 EntryList 再选）**只** **unpark** **这一个候选**（记录为 _succ）

- 被 unpark 后，它会先根据 succ 提示**短暂自旋**（期望减少与旁路线程的竞态），然后执行与自旋者同样的 CAS 获取流程：
  - 成功 ⇒ 清空 succ，成为新 owner；
  - 失败（被旁路线程插队） ⇒ 视策略再自旋一会儿或重新入队 park。

> 典型顺序感（并非严格保证）：**退出者唤醒一个等待者**，同时**仍允许自旋者插队**。这样能减少不必要的上下文切换，提高吞吐；代价是**不严格公平**，极端情况下可能出现短暂饥饿。

### #2.5.1

#### 可见性 / 原子性 / 有序性

- **可见性（Visibility）**

  一个线程的写，何时能被其他线程“看见”。JMM 通过 **happens-before（HB）边**来规定“写生效→读可见”的时机：

  - 同一变量的 **volatile 写 → 后续的 volatile 读** 建立 HB；
  - **解锁（monitorexit） → 后续对同一锁的加锁（monitorenter）** 建立 HB；
  - **线程启动/终止（start/join）**、**中断**、**final 字段发布**等也都建立特定 HB 边。

- **原子性（Atomicity）**

  单步、不可中断。JMM保证**单次**读/写是原子的（含 long/double 自 JDK5 起），但**复合操作**（x++、check-then-act、i=i+1）不是原子的，需要锁或原子类（CAS）来保证。

- **有序性（Ordering）**

  代码看起来的顺序，与实际执行/被其他线程观察到的顺序不一定一致。JMM允许**编译器/CPU**在不改变**单线程**语义的前提下**重排序**；多线程场景下，需靠 **HB** 边来**约束**这种重排。

```java
// 共享
Object data;
volatile boolean ready = false;

// 生产者
data = build();     // 普通写
ready = true;       // volatile 写（release）：禁止前面的写被移到它后面

// 消费者
if (ready) {        // volatile 读（acquire）：禁止后续读被移到它前面
    use(data);      // 能“看见”发布前对 data 的写
}
```

**JMM 语义**：对同一个 volatile 变量，**写 HB→ 读**。配合 release/acquire，使得 use(data) 一定能看到 build() 的效果。



synchronized 的 acquire/release

- **monitorenter****（加锁）= acquire**：禁止后续内存访问越到它之前；并保证能“看见”先前解锁方的写入。
- **monitorexit****（解锁）= release**：禁止之前的访问越到它之后；把本线程写入“对外发布”。
- 因此 **“解锁 HB→ 随后的加锁”**，临界区内的数据对后继持锁者**可见**且**有序**。

### #2.6.1

#### happens-before（HB）原则

**happens-before（HB）** 是 JMM 的“可见性与有序性”规则：若操作 A happens-before 操作 B，则 A 的所有写对 B 必然可见，且 B 不会把相关访问重排到 A 之前；若两者无 HB 关系，编译器/CPU 可自由重排、缓存，B 可能看不到 A 的结果。HB 由这些边构成：同线程的程序顺序、对同一锁的 **解锁→加锁**、同一 **volatile** 变量的 **写→读**、线程 **start/join/interrupt** 等，以及**传递性**（A→B 且 B→C ⇒ A→C）。实践结论：只要对共享数据的访问都被 HB 边覆盖（无数据竞争），程序行为等价于顺序一致执行（DRF-SC）。

### #2.7.1

#### volatile

volatile 的本质是**可见性 + 有序性**的承诺：对同一 volatile 变量，**写**相当于一次 *release*（在写点前的普通读写不得移动到它后面，并把本线程对共享数据的修改发布出去），**读**相当于一次 *acquire*（从内存/一致性层获取最新值，并禁止后续普通读写被提升到它前面），因此“volatile 写 **happens-before** 随后的 volatile 读”。HotSpot 通过在这些访问点插入合适的内存屏障/原子指令，并依赖 CPU 的缓存一致性协议（使其他核心对应缓存行失效）来实现这种语义。需要强调：volatile **不保证复合操作的原子性**（如 x++ 仍可能丢更新），它只保证**单次读/写的原子性**（含 long/double 自 JDK5 起）以及跨线程的可见性与有序性。

#### DLC（双重检查锁定）

```java
/**
 * DCL（双重检查锁定）懒汉式单例
 */
public final class Config {

    // 一定要是 volatile，防止指令重排与可见性问题
    private static volatile Config INSTANCE;

    private final String env;

    private Config() {
        this.env = "prod";
        // ... 其他一次性初始化
    }

    public static Config getInstance() {
        // 使用局部变量减少对 volatile 字段的重复读取
        Config local = INSTANCE;               // ① 读（可能为 null）
        if (local == null) {                   // 第一重检查（无锁快路径）
            synchronized (Config.class) {
                local = INSTANCE;              // ② 再读一次
                if (local == null) {           // 第二重检查（持锁慢路径）
                    local = new Config();      // ③ 构造
                    INSTANCE = local;          // ④ volatile 写（发布）
                }
            }
        }
        return local;                          // 返回已发布的对象
    }

    public String env() {
        return env;
    }
}
```

为什么需要 volatile

- 没有 volatile 时，new Config() 可能被重排为：分配内存 → 将引用赋给 INSTANCE → 执行构造函数。这样别的线程在第一重检查通过后，可能读到**尚未构造完成**的对象。
- volatile 的**写**（④）与随后其他线程的**读**共同建立 *happens-before*：构造函数中对字段的写在发布后对读线程可见，同时禁止了那种“先把引用写到 INSTANCE 再执行构造”的重排。

### #2.8.1

#### 原子类

- **CAS 内存语义**：一次 CAS 包含一次**volatile 读**（比较值）和——若成功——一次**volatile 写**（更新值）。因此它天然具备**可见性+有序性**（但不等价于互斥临界区）。
- **AtomicInteger/AtomicLong**：单热点计数在高并发下会出现**CAS 失败重试**风暴（都抢同一个变量）。

AtomicReference 这类“裸 CAS”没有版本号，天然会有 **ABA** 风险。做**链表**（或栈/队列）这类基于指针的无锁结构，常用这两类原子引用来兜底

AtomicStampedReference —— 带“版本号”的引用

```java
class Stack {
  static final class Node { final int v; final Node next; Node(int v, Node n){this.v=v; this.next=n;} }
  private final AtomicStampedReference<Node> head = new AtomicStampedReference<>(null, 0);

  void push(int v){
    int[] s = new int[1];
    Node h, n;
    do { h = head.get(s); n = new Node(v, h); }
    while (!head.compareAndSet(h, n, s[0], s[0] + 1));
  }
  Integer pop(){
    int[] s = new int[1];
    Node h;
    do { h = head.get(s); if (h == null) return null; }
    while (!head.compareAndSet(h, h.next, s[0], s[0] + 1));
    return h.v;
  }
}
```

- 每次 CAS 都把**stamp（版本）+1**，即便引用又回到旧值，也能检测到“中途被改过”。

AtomicMarkableReference —— 带“标记位”的引用

适合**Harris-Michael 链表**这类**逻辑删除 → 物理摘除**的算法：给 next 指针加一个 **mark**（已删标记），帮助并发删除并避免 ABA：

```java
static final class Node {
  final int key;
  final AtomicMarkableReference<Node> next = new AtomicMarkableReference<>(null, false);
}

// 删除时：CAS 把 (next, mark=false) → (next, mark=true) 逻辑删除
// 其他线程看见 mark=true 会“帮忙”把前驱.next 指到后继，完成物理摘除
```

- 这种“带标记的 next”配合**不重用节点对象**（不要对象池复用）可以很好地压住 ABA。
