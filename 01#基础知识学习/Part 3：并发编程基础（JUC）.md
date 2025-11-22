### #3.1.3

#### CompletableFuture

用“函数式管道”编排异步

**核心观念**

- CF 是一棵**依赖图**：每个 stage 表示一个结果的变换/组合；**不要阻塞 get()**，而是**用 thenXxx 链起来**。
- supplyAsync/runAsync 启动异步；**默认跑在** **ForkJoinPool.commonPool**（CPU 密集 OK，**阻塞 IO 不行**）。**遇到 IO 必须传自定义线程池**，否则会把 commonPool 饿死。
- thenApply（同步映射）、thenCompose（**扁平化**，把 Future<Future<T>> 变 Future<T>）、thenCombine/allOf（并行汇合）。
- 异常：exceptionally（兜底值）、handle（成败都处理）、whenComplete（不改结果的旁路观察）。
- 超时：JDK 9+ orTimeout/completeOnTimeout。

```java
var io = Executors.newFixedThreadPool(16);
var db = Executors.newFixedThreadPool(8);

// 谁先完成，谁的 thenAcceptAsync 就立刻去保存
CompletableFuture<Quote> p1 =
    CompletableFuture.supplyAsync(() -> getPriceByS1(), io);
CompletableFuture<Quote> p2 =
    CompletableFuture.supplyAsync(() -> getPriceByS2(), io);

//这里完全不阻塞，谁先 get 完谁就先执行 save，这里的 r 即为getPriceByS1()的返回参数Quote类本身
p1.thenAcceptAsync(q -> {
    // 1. 对 q 做一些处理（原地改 or 生成新对象都行）
    Quote processed = processQuote(q);  // 比如补充字段、计算折扣等等

    // 2. 再保存
    save(processed);
}, db);

p2.thenAcceptAsync(r -> save(r), db);

// 如果需要“在退出前等两次保存都结束”，再统一等一下（一次阻塞）
CompletableFuture.allOf(p1, p2).join();
```

关键点：

- **没有调用** **get()**；而是**把** **save** **作为回调挂上去**。谁先完成，谁先触发回调。
- 用 thenAcceptAsync(..., db) 指定保存跑在 **db 线程池**，避免把 IO 线程卡住。
- 只有你**确实需要**“所有保存都完成再返回”时，**最后统一** **allOf(...).join()** **一次**；而不是每个结果来一次 get() 阻塞。

> “非阻塞”的准确含义：**发起/编排的线程不被挂住**。真正干活的线程（例如执行 getPriceByS1() 的 IO 线程）当然会在 IO 上阻塞——这跟“非阻塞编排”不矛盾。

异常情况联动取消

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.function.Function;

public class PricePipeline {

    static class BusinessException extends RuntimeException {
        public BusinessException(String msg) { super(msg); }
        public BusinessException(String msg, Throwable cause) { super(msg, cause); }
    }

    // ========== 你的线程池 ==========
    static final ExecutorService io = Executors.newFixedThreadPool(16);
    static final ExecutorService db = Executors.newFixedThreadPool(8);

    // ========== 你的业务方法（示意）==========
    static Integer getPriceByS1() { /* ... 可能抛异常 ... */ return 100; }
    static Integer getPriceByS2() { /* ... 可能抛异常 ... */ return 101; }
    static void save(Integer r) { /* ... 可能抛异常 ... */ }

    // 早退检查（配合 fail-fast 标志 & 线程中断）
    static void checkCancelled(AtomicBoolean stop) {
        if (stop.get() || Thread.currentThread().isInterrupted()) {
            throw new CancellationException("stopped by peer failure");
        }
    }

    static Throwable unwrap(Throwable ex) {
        if (ex instanceof CompletionException || ex instanceof ExecutionException) {
            return ex.getCause() != null ? ex.getCause() : ex;
        }
        return ex;
    }

    public static void main(String[] args) {
        AtomicBoolean stop = new AtomicBoolean(false);

        // 保存阶段统一封装：在 DB 线程池执行，开始前先看是否已被停止
        Function<Integer, CompletableFuture<Void>> saveAsync = (r) ->
            CompletableFuture.runAsync(() -> {
                checkCancelled(stop);   // 同伴失败时，跳过 save
                save(r);
            }, db);

        // 链路1：get(S1) -> save
        CompletableFuture<Void> s1 =
            CompletableFuture.supplyAsync(() -> {
                checkCancelled(stop);
                return getPriceByS1();
            }, io).thenComposeAsync(saveAsync, db);

        // 链路2：get(S2) -> save
        CompletableFuture<Void> s2 =
            CompletableFuture.supplyAsync(() -> {
                checkCancelled(stop);
                return getPriceByS2();
            }, io).thenComposeAsync(saveAsync, db);

        // 失败联动：任一链路异常 -> 标记停止并尝试取消同伴
        s1.whenComplete((v, ex) -> {
            if (ex != null) {
                stop.set(true);
                s2.cancel(true); // CompletableFuture 的取消不一定能打断正在运行的任务，但能阻止后续阶段
            }
        });
        s2.whenComplete((v, ex) -> {
            if (ex != null) {
                stop.set(true);
                s1.cancel(true);
            }
        });

        // 一次阻塞：等两条链路都“完成”（成功/异常/取消其一都会让 allOf 异常完成）
        try {
            CompletableFuture.allOf(s1, s2).join();
            // 全部成功才会到达这里
        } catch (CompletionException | CancellationException e) {
            throw new BusinessException("price pipeline failed", unwrap(e));
        } finally {
            // 视需要决定是否关闭线程池（多数服务内线程池是全局复用，不要随意关闭）
            // io.shutdown();
            // db.shutdown();
        }
    }
}
```



#### 虚拟线程

JDK 21 的虚拟线程本质是 JVM 级“线程”：每个虚拟线程都有一个 **Continuation**，把其调用栈切成若干 **stack chunks** 存在堆上；运行时由少量平台“承载线程”（carrier，默认是一个轻量调度器/工作窃取池）**挂载（mount）**到某个 carrier 上执行。遇到“可挂起点”（LockSupport.park/sleep、wait/join、Condition、以及“虚拟线程感知”的阻塞 I/O 等），JVM/类库把操作转为 **非阻塞 I/O + 事件回调**，并在用户态做 **卸载（unmount）**：保存续体、释放 carrier，不占用 OS 线程；待事件就绪或被 unpark，再把该虚拟线程 **重新挂载** 到某个 carrier 恢复执行。因此能以极低成本承载海量“阻塞式”代码。若在 synchronized 临界区或本地（native）调用里阻塞，栈会被 **pin**（钉住），carrier 无法释放，伸缩性下降，这是主要注意点。整体保留 Thread/调试/栈跟踪语义，但调度是协作式的“保存-恢复栈帧 + 任务重排”，实现了“写同步代码，享异步伸缩”。

- **stack chunks（栈分片）**：虚拟线程的调用栈不是固定大小的原生线程栈，而是被切成若干个小块（chunk）放在**堆**里，按需增长/回收。这样一旦需要挂起，JVM可以把这些栈块当成对象保存起来，等恢复时再把它们装回去继续执行——这就是“可保存/可恢复的栈（continuation）”的基础。
- **carrier（承载线程）**：真正的**OS 线程**，用来执行虚拟线程的一小段运行片段。一个进程里有少量 carrier，它们像一个调度器的工作线程池，轮流“托管”成千上万的虚拟线程。
- **mount / unmount（挂载/卸载）**：当某个虚拟线程要跑，它会被**挂载（mount）到某个 carrier 上，开始执行；遇到可挂起点（比如 park、sleep、等待 I/O 就绪）时，JVM把它的栈块保存到堆里，把它卸载（unmount）下来，释放 carrier 去干别的；事件就绪后再把该虚拟线程重新挂载**到某个（不一定是原来的）carrier 继续从断点往下跑。

```
VT 挂到某个 carrier 上执行 → 调用 socket.read()（表面阻塞）
   └─ JDK 把 Socket 置非阻塞 + 在 Selector 注册可读
      └─ VT park → unmount（carrier 释放去干别的）
         └─ epoll/kqueue 告知可读 → 调度器 remount VT 到某个 carrier
            └─ 从 read 返回点继续执行（你的代码无感）
```

所以，总结一句话：**虚拟线程靠“堆上可保存的栈分片 + 少量承载线程 + 挂载/卸载调度”，把传统阻塞式网络 I/O 在库层改造成事件驱动/非阻塞执行，让你写同步代码但不占着 OS 线程；碰到不能卸载的“pin”场景才会退化成占用 carrier。**

```java
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<Integer> f1 = exec.submit(() -> { int r = getPriceByS1(); save(r); return r; });
    Future<Integer> f2 = exec.submit(() -> { int r = getPriceByS2(); save(r); return r; });
    // 需要等都完成就 join 一次；不等也可以直接返回
    f1.get(); f2.get();
}
```

### #3.3.1

#### synchronized-ReentrantLock

**synchronized 的工作原理（1 段）**

synchronized 在字节码里是 monitorenter/monitorexit 成对出现：进入时先走**轻量级锁**快路径——在当前线程栈压入 **lock record**，用 **CAS** 把对象头的 **Mark Word** 改成指向该记录；重入再压一层记录即可。竞争激烈、需要 wait/notify 或与 identityHashCode 冲突时会**膨胀为重量级 ObjectMonitor**：维护 owner/recursions、EntryList/WaitSet，失败方先**自适应自旋**，再 park，释放方将 owner=null 并 unpark 一名候选者竞争。内存语义上：monitorenter 是 **acquire**，monitorexit 是 **release**，保证临界区写在解锁后对后继加锁者**可见且有序**；异常路径编译器会生成 finally 以确保 monitorexit 必执行。



**ReentrantLock 的工作原理（1 段）**

ReentrantLock 基于 **AQS（AbstractQueuedSynchronizer）** 的独占模式：用一个 volatile int state 表示持有计数，tryAcquire 通过 **CAS** 将 state:0→1 并设置 exclusiveOwnerThread；失败则把线程封装成节点加入 **CLH/FIFO 同步队列**，前驱未释放前**自旋片刻→park**；释放时 state--，到 0 清空 owner 并 unpark 后继。它支持**可重入**、**公平/非公平**两种策略、**可中断/可定时**获取（lockInterruptibly/tryLock(timeout)），以及多路 **Condition**（await/signal）各自拥有独立等待队列。AQS 在成功获取/释放处插入 **acquire/release** 语义，提供与 synchronized 等价的可见性/有序性保障。



**主要区别（1 段）**

语义层面两者都**可重入且提供 HB 的互斥**，但：synchronized 是**语言级关键字**，语法简单、异常自动释放（编译器保证 monitorexit），不支持**中断/定时/公平**与多条件队列；ReentrantLock 是**库级锁**，需手写 try/finally 保证 unlock，但提供**可中断/超时/公平策略**与**多个 Condition**、tryLock/可观测方法（队列长度、是否持有等），可精细调度。在性能上，现代 JVM 的 synchronized（轻锁+自旋+膨胀）已很强，低争用下与 ReentrantLock 接近；高争用与复杂等待模型时常选 ReentrantLock。另外对 **虚拟线程**：在临界区内阻塞 I/O 时，synchronized 会**pin** 载体线程，而基于 LockSupport.park/unpark 的 ReentrantLock/Condition 更**虚拟线程友好**（可挂起而不占用 OS 线程）。

### #3.4.1

#### ReentrantReadWriteLock

下面把 **ReentrantReadWriteLock（RRWL）** 讲清楚：工作原理 → 语义与用法 → 常见坑（含降级/不支持升级、条件变量等）。



工作原理（基于 AQS）

- **两把锁**：ReadLock（共享模式）和 WriteLock（独占模式），共用一个 AQS 同步器。

- **状态位设计**：AQS 的 state（32 位）被拆成两部分——

  - 低 16 位：**写锁重入计数**（仅持有写锁的线程可递增）。

  - 高 16 位：**读锁持有总数**（所有读线程计数之和）。

    线程本地还维护**每线程读计数**（HoldCounter，带首读者优化 firstReader 与 cachedHoldCounter）以支持读锁的可重入与释放。

- **获取规则**：

  - **写锁**：无读者且无其他写者时，CAS 抢占；同线程可**重入**（写计数+1）。
  - **读锁**：在**没有他人持有写锁**或**只有当前线程重入写锁**时可获取；读锁本身也**可重入**（每线程读计数+1）。

- **偏向写者（避免写饥饿）**：一旦有**等待写者**，后来的读者通常会被挡在门外（除非是持有写锁的线程自己重入读锁），以免写线程长期挨饿。

- **中断/超时**：支持 lockInterruptibly() 和 tryLock(timeout)；tryLock() 为非阻塞尝试。

- **内存语义**：与 Lock 一致，成功的 unlock 对后续成功的 lock 建立 **happens-before**。特别是**写解锁 → 之后任一（读/写）加锁**建立 HB。



语义与用法要点

- **读多写少**场景提升吞吐：多个读者可**并发**进入，写者**互斥**且与读者互斥。
- **可重入**：
  - 写锁：同线程可重入（计数在低 16 位）。
  - 读锁：同线程可重入（线程本地计数）。
- **降级（支持）**：**写→读**（常用于“修改后继续读”保持视图一致）：

```java
rw.writeLock().lock();
try {
    update();
    rw.readLock().lock();   // 先拿读锁
} finally {
    rw.writeLock().unlock(); // 再放写锁 —— 完成降级
}
try {
    readView();             // 仍受读锁保护
} finally {
    rw.readLock().unlock();
}
```

- **升级（不支持）**：**读→写**会死锁风险（自己拿着读锁挡住写锁），RRWL**没有**安全升级通道；需要写入时应**先释放所有读锁**再尝试写锁，或改用 StampedLock.tryConvertToWriteLock。
- **公平/非公平**：构造时可选；非公平允许“插队”以提高吞吐，公平降低抖动但吞吐偏低。
- **条件变量**：**只有写锁**支持 newCondition()；读锁不支持 Condition（会抛 UnsupportedOperationException）。



什么时候用 / 什么时候别用

- **用它**：读占比高、读操作可并发且**读持有时间短**、写不频繁（例如配置快照、缓存元数据、索引查阅）。
- **谨慎/别用**：写多/写持有时间长、读操作很短（上下文切换不划算），或需要读→写**无缝升级**（考虑 StampedLock 或细化分段锁）。



常见坑与提示

- **误升级**：在持有读锁时 writeLock().lock() 可能自锁住自己——先释放读锁再获取写锁。
- **长读锁=压制写者**：虽然偏向写者，但已有读者没释放前写者始终进不来；避免把 IO/耗时操作放在读锁内。
- **虚拟线程友好**：基于 park/unpark 的 AQS，阻塞时可挂起而不占用 OS 线程（比 synchronized 更 VT-friendly）。



快速模板

```java
var rw = new ReentrantReadWriteLock();
var r  = rw.readLock();
var w  = rw.writeLock();

// 只读
r.lock();
try { readOnly(); } finally { r.unlock(); }

// 写（含降级）
w.lock();
try {
    update();
    r.lock();      // 降级
} finally {
    w.unlock();
}
try { readAfterUpdate(); } finally { r.unlock(); }
```

> 总结：RRWL 用 AQS 把“读共享/写独占”做成**可重入、可中断、可定时**的锁，**支持降级不支持升级**，且默认**避免写饥饿**。读多写少、读区短小的场景收益显著。

#### StampedLock

- **三种模式**：WRITE（独占）、READ（共享）、**乐观读**（OPTIMISTIC，无加锁，失败可回退）。
- **票据 stamp（long）**：任何获取都会返回一个 long；释放或转换必须带回这个 stamp。**0 表示失败/无效**。
- **非可重入**：同一线程**不能**在持有写锁时再次 writeLock()，也不能在持有读锁时再次 readLock()（会自锁）。改用**转换**。
- **无公平保证**：可能出现读或写短暂饥饿。
- **中断/超时**：无参的 readLock()/writeLock() **不可中断**；try*Lock(timeout, unit) **可中断**并可能抛 InterruptedException。
- **内存语义**：与 Lock 一致——成功获取 = **acquire**，成功释放 = **release**；乐观读**只有在** **validate(stamp)** **成功时**才视为看到了“稳定快照”。



乐观读（优先选它，失败再回退）

```java
long s = sl.tryOptimisticRead(); // 非阻塞，拿个快照
Data d1 = this.d1;              // 直接读共享数据
int  d2 = this.d2;
if (!sl.validate(s)) {          // 有写发生过 → 回退到读锁
    s = sl.readLock();
    try { d1 = this.d1; d2 = this.d2; }
    finally { sl.unlockRead(s); }
}
// 使用 d1,d2
```



正常读写锁 

```java
// 读
long rs = sl.readLock();
try { readOnly(); }
finally { sl.unlockRead(rs); }

// 写
long ws = sl.writeLock();
try { mutate(); }
finally { sl.unlockWrite(ws); }
```



升级/降级（StampedLock支持“尝试转换”）

- **读→写（升级）**：先尝试原地升级，失败再释放读锁并获取写锁（注意竞态）。

```java
long s = sl.readLock();
try {
    if (!condition()) return;
    long ws = sl.tryConvertToWriteLock(s);
    if (ws == 0L) {             // 升级失败：释放读锁再申请写锁
        sl.unlockRead(s);
        ws = sl.writeLock();    // 这里可能阻塞
    }
    s = ws;                     // 现在持有写锁
    update();
} finally {
    sl.unlock(s);               // unlock 可解任意模式
}
```



- **写→读（降级）**：在写锁内直接转读锁，避免“写完立刻放开导致读窗口暴露”。

```java
long s = sl.writeLock();
try {
    update();
    s = sl.tryConvertToReadLock(s); // 一般会成功
    // 持有读锁，继续读视图
    view();
} finally {
    sl.unlock(s);
}
```



与 ReentrantReadWriteLock 的对比（一句话）

- RRWL：可重入、支持 Condition、不支持读→写**升级**（易自锁），没有乐观读。
- **StampedLock**：**非可重入**、无 Condition、提供**乐观读**与**原地转换**（tryConvert…），在读多写少的场景通常**吞吐更高、延迟更稳**。



常见坑

- **忘记 validate**：乐观读之后**必须** validate(s)，否则读到“撕裂”数据。
- **误用为重入锁**：再次 readLock()/writeLock() 会自己卡住；要用 tryConvert…。
- **长读区/阻塞操作**：会拖慢写并增加乐观读失败率；把 IO/重活移出锁内。
- **中断语义**：需要可中断获取时，用带超时的 try*Lock(timeout, unit) 版本。
- **虚拟线程**：内部基于 park/unpark，对 VT 友好；但**在写锁临界区内做长阻塞**仍不建议（会放大争用）。



### #3.5.1

#### AQS、CLH

**AQS（AbstractQueuedSynchronizer）**：J.U.C 的底层同步框架，核心是一个 volatile int state（由子类用 CAS 在 tryAcquire/tryRelease 或共享版 tryAcquireShared/tryReleaseShared 中定义语义）和一个基于 CLH 思想的 **FIFO 双向等待队列**。获取失败的线程被封装为 Node 入队，前驱为 head 才有资格再次尝试，否则用 LockSupport.park() 阻塞，释放时由持有者 unpark 唤醒后继。它支持**独占/共享**两种模式（如 ReentrantLock vs Semaphore/CountDownLatch）、**公平/非公平**策略、**可重入**计数，以及与同步队列分离的 **Condition** 条件队列（await/signal 通过在两队列间转移节点实现）。



**CLH（Craig–Landin–Hagersten）队列锁**：一种**自旋**队列锁。每个线程在入场时新建/复用本地 Node{locked=true}，用原子交换把自己设为队尾并拿到**前驱节点**指针，然后**只在本地自旋观察前驱节点的状态**（前驱将 locked=false 表示释放）。由于自旋仅命中少量共享缓存行，CLH 具有良好的缓存亲和与 FIFO 公平性；实现简单、开销小，适合**临界区很短、期望快速交接**的场景。其亲属 MCS 锁改为自旋在“自己的节点”。



**区别（一句话归纳）**：AQS 是“**阻塞型**的通用同步框架”：显式**双向链表**排队、park/unpark 省 CPU、支持**独占/共享、条件队列、可重入与公平策略**，适合**竞争激烈或等待可能较长**的场景；CLH 是“**自旋型**的互斥锁算法”：**隐式单向队列**、线程**在前驱上本地自旋**、天然 FIFO 公平但**不自带条件/共享/重入**（需额外封装），适合**临界区极短、希望极低切换开销**的场景。换句话说：**AQS=通用、可阻塞；CLH=轻量、自旋**。



### #3.6.1

#### ArrayBlockingQueue（ABQ）

- **特性**：固定容量、**数组**存储、严格 FIFO、内存紧凑；可选 **公平**（new ArrayBlockingQueue(cap, true)）。
- **实现**：**一把锁**（ReentrantLock）+ 两个条件（notEmpty/notFull）；入/出都争同一把锁。
- **适用**：**稳定速率**的生产/消费（限流、背压），对内存占用可预估。
- **坑点**：高并发下入/出同锁易形成**锁竞争**；公平模式降低吞吐。



#### LinkedBlockingQueue（LBQ）

- **特性**：**链表**节点；构造不传容量则**近似无界**（实际受内存限制）；设上限才适合生产。
- **实现**：**两把锁**（putLock/takeLock）+ 两个条件，入/出在高并发下更易并行；size() 代价高且仅近似值。
- **适用**：生产/消费**吞吐**要求较高、生产和消费都很活跃；**一定要**指定容量，如 new LinkedBlockingQueue<>(100_000).
- **坑点**：默认无界 → **OOM 隐患**；每个节点额外对象，**内存开销大**、GC 压力高。



#### SynchronousQueue

- **特性**：**容量=0**，不存货；生产者与消费者**当面交接（handoff）**。
- **实现**：两种内部结构：非公平（栈，LIFO，默认）/公平（队列，FIFO）；使用 park/unpark 配对。
- **适用**：**线程/池之间切换**、**一手交钱一手交货**的模型；例如 newCachedThreadPool 就用它。
- **坑点**：offer() 只有在**有消费者等待**时才成功；否则要用 offer(timeout) 或 put()（会阻塞）。无缓冲 → 容易出现**全局阻塞**。



#### DelayQueue / PriorityBlockingQueue（PBQ）

- **DelayQueue**

  - **特性**：基于小顶堆的**延时队列**，元素实现 Delayed；take() 只会取出**到期**元素。
  - **实现**：内部是 PriorityQueue + 条件等待；以 System.nanoTime() 计算剩余延迟。
  - **适用**：定时/延时任务（重试、TTL）。
  - **坑点**：**无界**；未到期元素会一直占内存；drainTo 只会**拉走已到期**的元素。

  

- **PriorityBlockingQueue**

  - **特性**：**按优先级**出队（最小/最高优先），逻辑**无界**（构造容量只是初始堆大小）。
  - **实现**：PriorityQueue + 可重入锁；插入永不阻塞，只有取空时阻塞。
  - **适用**：调度按权重/优先级的任务。
  - **坑点**：**不能真正限流**（生产永不阻塞）；比较器要**稳定且快速**，否则放大 CPU。

  

API 语义速查（阻塞 vs 非阻塞）

| **操作**          | **满/空时行为**                                              |
| ----------------- | ------------------------------------------------------------ |
| add(e)            | 满则**抛异常** IllegalStateException（LBQ/PBQ/DelayQueue 由于“无界”，几乎不会抛） |
| offer(e)          | 满则**返回 false**（PBQ/DelayQueue 通常一直 true）           |
| offer(e, t, unit) | 等待至多 t，超时**返回 false**                               |
| put(e)            | **阻塞**直到有空间（SynchronousQueue 需对端已等待）          |
| remove()          | 空则**抛异常** NoSuchElementException                        |
| poll()            | 空**返回 null**（DelayQueue：仅到期元素可取）                |
| poll(t, unit)     | 等待至多 t；超时返回 null                                    |
| take()            | **阻塞**直到有元素（DelayQueue：**到期**元素）               |
| peek()/element()  | 非破坏性取头；空时 peek=null / element 抛异常                |
| drainTo(c[, max]) | 批量出队；DelayQueue 只会**转移已到期**的                    |

> 生产上常用 offer(e, timeout) 来**背压**，避免生产端无限堆积。



选型建议（一句话版）

- **固定速率、可预估内存** → **ABQ**（数组、有界、紧凑）。
- **高并发吞吐、入/出都很忙** → **LBQ(有界)**（两锁并行、注意内存）。
- **必须一手交接、不要缓冲** → **SynchronousQueue**（handoff）。
- **定时/延迟** → **DelayQueue**（配合到期检查/drainTo）；
- **优先级调度**、不需要限流 → **PriorityBlockingQueue**（注意比较器与“无界”）。



两个实用小范式

**1）生产端带背压，消费端阻塞取**

```
var q = new ArrayBlockingQueue<Task>(10_000);
boolean ok = q.offer(task, 200, TimeUnit.MILLISECONDS);
if (!ok) { log.warn("queue busy, drop or fallback"); }
```

**2）SynchronousQueue 线程切换（谁先来谁等谁）**

```
var q = new SynchronousQueue<Job>();
// producer
exec.submit(() -> { q.put(loadJob()); }); // 阻塞直到有消费者 take
// consumer
exec.submit(() -> { Job j = q.take(); handle(j); });
```



常见易错点

- **LBQ 不设容量** = 隐形炸弹（OOM 事故高发）。
- **PBQ/DelayQueue 不能限流**：生产端不会被阻塞。
- **ABQ 公平模式**吞吐更低，仅在强公平需求时用。
- **SynchronousQueue** 在两端未对齐时会**全面阻塞**；适合作为“线程切换点”，不是缓冲器。
- **DelayQueue** **drainTo** 只能拿走**到期**元素，别指望一次性清空未来任务。



### #3.7.1

#### CountDownLatch

CountDownLatch 是一个**一次性**倒计时闩锁。

- 初始化一个**计数值 n**。
- 多个线程调用 countDown() 使计数减 1（**不阻塞**）。
- 线程调用 await() **阻塞等待**，直到计数减到 0；之后所有后续 await() 都将**立刻返回**。
- **不可重置**（one-shot），要复用只能新建一个。



核心 API

- new CountDownLatch(int n)：n ≥ 0；n=0 时 await() 立即返回。
- void countDown()：减到 0 时**唤醒所有等待者**；到 0 后继续调用无效果、不会变负数。
- void await()/boolean await(long, TimeUnit)：可被**中断**；带超时的返回 false 表示超时未到 0。



Happens-Before（非常关键）

Javadoc 明确：**任一线程在 countDown() 之前的动作，对另一个成功从 await() 返回的线程是可见的**（建立 *happens-before*）。

这保证了“生产者设置数据 → countDown() → 消费者 await() 返回后读取数据”的**可见性与有序性**。

示例（安全发布结果）：

```java
class Demo {
    private final CountDownLatch ready = new CountDownLatch(1);
    private volatile String result; // volatile 可省，但经由 latch 也已安全可见

    void producer() {
        result = compute();  // 1) 写入结果
        ready.countDown();   // 2) 发布点（HB: 1 -> await-return）
    }

    String consumer() throws InterruptedException {
        ready.await();       // 返回后能看见 producer 对 result 的写
        return result;
    }
}
```



典型用法模式

1) 等一批任务全部完成再继续（线程池里最常见）

```java
int n = tasks.size();
CountDownLatch done = new CountDownLatch(n);
for (Runnable t : tasks) {
    pool.execute(() -> {
        try {
            t.run();
        } finally {
            done.countDown(); // 一定放 finally，出错也要减计数，防“永远卡住”
        }
    });
}
done.await(); // 全部完成再往下
```



2) “一起开跑 + 一起结束”的基准/压测

```java
CountDownLatch start = new CountDownLatch(1);
CountDownLatch end   = new CountDownLatch(nThreads);

for (int i = 0; i < nThreads; i++) {
    pool.execute(() -> {
        try {
            start.await();   // 等统一起跑
            work();
        } finally {
            end.countDown(); // 报告完成
        }
    });
}

start.countDown(); // 发令枪
end.await();       // 等所有选手跑完
```



3) 把回调式异步“桥接”为同步等待

```java
CountDownLatch latch = new CountDownLatch(1);
AtomicReference<Response> ref = new AtomicReference<>();
client.asyncCall(req, resp -> { ref.set(resp); latch.countDown(); },
                      err -> { ref.set(null);  latch.countDown(); });

if (!latch.await(500, TimeUnit.MILLISECONDS)) { /* 超时处理 */ }
Response resp = ref.get();
```



常见坑 & 最佳实践

- **finally 里** **countDown()**：任何异常都必须保证减计数，否则等待方会**永远卡住**。
- **await 可被中断**：await() 抛出 InterruptedException 会**清除中断位**；调用方应**恢复中断或向上抛出**（Thread.currentThread().interrupt()），避免“吞中断”。
- **不可重置**：需要多轮或动态参与者，请用 **CyclicBarrier**（可重用、让“一批线程互等”）或 **Phaser**（参与方可增减，场景更灵活）。
- **别把它当互斥锁**：CountDownLatch 不是锁，不用于保护临界区。
- **计数要匹配**：初始化的 n 必须与实际 countDown() 次数匹配；过多 countDown() 没副作用，但**过少会死等**。
- **与** **CompletableFuture** **的取舍**：如果已是 CF 流水线，优先 allOf()/anyOf()；CountDownLatch 更适合**与非 CF/回调式 API**对接，或做“一次性门闩”。



与其他同步工具的区别

- CountDownLatch：**一次性**、等待“计数归零”；多等待者被**一次性全放行**。
- CyclicBarrier：**可循环复用**，参与者数固定；每轮所有人都到达栅栏后**同时继续**。
- Phaser：可动态注册/到达/撤销，适合多阶段、参与者变化的流程。
- Semaphore：配额/限流。
- Future/CF：面向“任务完成/结果传播”的抽象，很多时候更自然。



一个结合你上个话题的“失败即早退”范式

如果用 CountDownLatch 等两条任务，但**任一失败就放弃另一条**，可以配合中断/标志位：

```java
CountDownLatch done = new CountDownLatch(2);
AtomicBoolean stop = new AtomicBoolean(false);
AtomicReference<Throwable> firstErr = new AtomicReference<>();

pool.execute(() -> {
    try {
        if (!stop.get()) getAndSave1();
    } catch (Throwable e) {
        if (firstErr.compareAndSet(null, e)) stop.set(true);
    } finally { done.countDown(); }
});
pool.execute(() -> {
    try {
        if (!stop.get()) getAndSave2();
    } catch (Throwable e) {
        if (firstErr.compareAndSet(null, e)) stop.set(true);
    } finally { done.countDown(); }
});

done.await(); // 或 await(timeout)
Throwable err = firstErr.get();
if (err != null) throw new BusinessException("pipeline failed", err);
```

> 说明：Latch 本身不提供取消，需用共享标志（或线程中断+可中断阻塞）协作“早退”。若你的任务已基于 CompletableFuture，更推荐用 anyOf/allOf + 异常完成来表达。



例子：并发调用外部接口，**等所有结果都返回后**在主线程里做汇总

future 阻塞 get

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class BatchCallWithThreadPool {

    // 假设外部接口返回的类型
    static class Quote {
        private final String source;
        private final double price;
        Quote(String source, double price) { this.source = source; this.price = price; }
        public String getSource() { return source; }
        public double getPrice() { return price; }
        @Override public String toString() { return source + ":" + price; }
    }

    // 模拟外部调用方法（真实项目里就是你要调的接口）
    static Quote callExternal(String source) throws Exception {
        // 模拟耗时
        TimeUnit.MILLISECONDS.sleep(200 + ThreadLocalRandom.current().nextInt(300));
        // 模拟价格
        return new Quote(source, 100 + ThreadLocalRandom.current().nextDouble(50));
    }

    public static void main(String[] args) throws Exception {
        ExecutorService io = Executors.newFixedThreadPool(8);

        try {
            List<String> sources = Arrays.asList("S1", "S2", "S3", "S4", "S5");

            // 1) 组装一批 Callable 任务（每个任务调用一次外部接口）
            List<Callable<Quote>> tasks = new ArrayList<>();
            for (String s : sources) {
                tasks.add(() -> {
                    return callExternal(s);
                });
            }

            // 2) 并发执行，invokeAll 会阻塞直到“所有任务都完成”
            List<Future<Quote>> futures = io.invokeAll(tasks);

            // 3) 主线程统一收集结果（这里 futures 都已完成，get() 只会很快返回/抛异常）
            List<Quote> quotes = new ArrayList<>(futures.size());
            for (Future<Quote> f : futures) {
                try {
                    quotes.add(f.get());
                } catch (ExecutionException e) {
                    // 有任一失败：按需处理，这里直接抛出去
                    throw new RuntimeException("有外部调用失败", e.getCause());
                }
            }

            // 4) 汇总：例如找最低价、计算平均价
            Quote best = null;
            double sum = 0.0;
            for (Quote q : quotes) {
                if (best == null || q.getPrice() < best.getPrice()) best = q;
                sum += q.getPrice();
            }
            double avg = sum / quotes.size();

            System.out.println("最便宜：" + best);
            System.out.println("平均价：" + avg);
        } finally {
            io.shutdown();
        }
    }
}
```

CountDownLatch 写法

```java
import java.util.*;
import java.util.concurrent.*;

public class BatchCallWithLatch {
    // 模拟外部接口的返回类型
    static class Quote {
        final String source; final double price;
        Quote(String s, double p) { source = s; price = p; }
        @Override public String toString() { return source + ":" + price; }
    }

    // 模拟外部调用
    static Quote callExternal(String source) throws Exception {
        TimeUnit.MILLISECONDS.sleep(200 + ThreadLocalRandom.current().nextInt(300)); // 耗时
        return new Quote(source, 100 + ThreadLocalRandom.current().nextDouble(50));
    }

    public static void main(String[] args) throws Exception {
        ExecutorService io = Executors.newFixedThreadPool(8);

        List<String> sources = Arrays.asList("S1","S2","S3","S4","S5");
        int n = sources.size();

        CountDownLatch done = new CountDownLatch(n);
        Quote[] results = new Quote[n];                                // 用数组装结果，简洁且线程安全地按下标写入
        ConcurrentLinkedQueue<Throwable> errors = new ConcurrentLinkedQueue<>();

        for (int i = 0; i < n; i++) {
            final int idx = i;
            final String s = sources.get(i);
            io.execute(() -> {
                try {
                    Quote q = callExternal(s);
                    results[idx] = q;                                  // 写结果
                } catch (Throwable e) {
                    errors.add(new RuntimeException("调用 " + s + " 失败", e));
                } finally {
                    done.countDown();                                  // 一定放 finally，防止永远卡住
                }
            });
        }

        // 等全部任务结束（也可以用 done.await(3, TimeUnit.SECONDS) 做整体超时）
        done.await();

        io.shutdown(); // 可选：优雅关闭

        // 若有失败，按需处理（这里直接抛出第一个）
        if (!errors.isEmpty()) {
            throw new RuntimeException("有外部调用失败", errors.peek());
        }

        // 主线程做汇总
        Quote best = null;
        double sum = 0.0;
        for (Quote q : results) {
            if (best == null || q.price < best.price) best = q;
            sum += q.price;
        }
        double avg = sum / results.length;

        System.out.println("最便宜：" + best);
        System.out.println("平均价：" + avg);
    }
}
```

CompletableFuture

```java
import java.util.*;
import java.util.concurrent.*;

public class BatchCallWithCompletableFuture {

    // 模拟外部接口的返回类型
    static class Quote {
        final String source; final double price;
        Quote(String s, double p) { source = s; price = p; }
        @Override public String toString() { return source + ":" + price; }
    }

    // 模拟外部调用
    static Quote callExternal(String source) throws Exception {
        TimeUnit.MILLISECONDS.sleep(200 + ThreadLocalRandom.current().nextInt(300)); // 耗时
        return new Quote(source, 100 + ThreadLocalRandom.current().nextDouble(50));
    }

    public static void main(String[] args) {
        ExecutorService io = Executors.newFixedThreadPool(8);

        try {
            List<String> sources = Arrays.asList("S1","S2","S3","S4","S5");

            // 1) 为每个来源启动一个异步任务
            List<CompletableFuture<Quote>> cfs = new ArrayList<>();
            for (String s : sources) {
                CompletableFuture<Quote> cf =
                    CompletableFuture.supplyAsync(() -> {
                        try {
                            return callExternal(s);
                        } catch (Exception e) {
                            throw new CompletionException(e);
                        }
                    }, io)
                    // 可选：单任务超时（JDK9+）
                    .orTimeout(2, TimeUnit.SECONDS);
                cfs.add(cf);
            }

            // 2) 等所有任务完成（如果任一失败/超时，这里会抛 CompletionException）
            try {
                CompletableFuture
                    .allOf(cfs.toArray(new CompletableFuture[0]))
                    .join();
            } catch (CompletionException e) {
                // 可选：失败时取消其他未完成任务，避免浪费线程
                for (CompletableFuture<?> cf : cfs) {
                    if (!cf.isDone()) cf.cancel(true);
                }
                throw new RuntimeException("有外部调用失败", e.getCause());
            }

            // 3) 主线程统一收集结果并汇总
            List<Quote> quotes = new ArrayList<>(cfs.size());
            for (CompletableFuture<Quote> cf : cfs) {
                quotes.add(cf.join()); // 此时都已完成，join 很快返回
            }

            Quote best = null;
            double sum = 0.0;
            for (Quote q : quotes) {
                if (best == null || q.price < best.price) best = q;
                sum += q.price;
            }
            double avg = sum / quotes.size();

            System.out.println("最便宜：" + best);
            System.out.println("平均价：" + avg);
        } finally {
            io.shutdown();
        }
    }
}
```



### #3.9.1

#### ThreadLocal

- **本质**：ThreadLocal 把“变量值”挂在**线程对象**里的 ThreadLocalMap 上（key 为 ThreadLocal 的弱引用，value 是强引用）。同一 ThreadLocal 在不同线程各有一份独立副本。
- **为什么会“泄漏”**：线程池里的工作线程**长期存活**，如果业务代码只 set 不 remove，值会一直挂在线程上；哪怕 ThreadLocal 自己被回收（key= null），value 仍可能残留，直到 ThreadLocalMap 触发清理或线程结束。因此**在线程池场景务必** **finally { tl.remove(); }**。

```java
static final ThreadLocal<Ctx> CTX = new ThreadLocal<>();

void handle() {
    CTX.set(new Ctx(...));
    try {
        // ... 业务逻辑
    } finally {
        CTX.remove(); // 线程池下必须清!
    }
}
```



InheritableThreadLocal 的“继承”与线程池陷阱

- **语义**：新建子线程时把父线程的值**拷贝**给子线程。

- **线程池问题**：池中线程不是“刚创建”，而是复用旧线程，所以**不会再次拷贝**；导致要么拿不到期望的上下文，要么残留旧上下文。

  结论：在**线程池/异步**里不要指望 InheritableThreadLocal。若确实需要上下文传递，使用“捕获→包装→恢复”的执行器装饰器（见 §5）。



Spring 里哪些用到了 ThreadLocal

1. **事务&资源绑定（核心）**

   - 入口：TransactionSynchronizationManager（TSM）。内部用 ThreadLocal 维护：
     - resources（如 DataSource -> ConnectionHolder、SessionFactory -> SessionHolder）
     - synchronizations（回调列表）
     - currentTransactionName / readOnly / isolationLevel / actualTransactionActive
   - JDBC：DataSourceTransactionManager 开启事务时获取 Connection，包成 ConnectionHolder 并 bindResource(dataSource, holder)；之后同线程内任何 DataSourceUtils.getConnection(ds) 都会复用该连接；提交/回滚时通过同步回调释放并 unbind。
   - JPA/Hibernate：在事务或 OpenEntityManagerInView/OpenSessionInView 过滤器中，把 EntityManager/Session 绑定到线程，同理复用与清理。

   

2. **请求/会话上下文**

   - RequestContextHolder：将 ServletRequestAttributes 绑定到线程，使非控制器代码也能拿到请求对象/会话属性（例如 LocaleContextHolder 也常配合它工作）。
   - LocaleContextHolder：当前线程的 Locale/TimeZone。
   - DateTimeContextHolder（Spring 6+）等：与时间格式化上下文相关。

   

3. **安全上下文**

   - SecurityContextHolder：默认使用 ThreadLocal 保存 SecurityContext（当前认证用户信息）。在 @Async 或手动线程池中需使用 DelegatingSecurityContext* 包装 Executor/Runnable/Callable 来传播。

   

4. **日志上下文（非 Spring 核心但常配）**

   - MDC（Logback/Log4j2）：本质也是 ThreadLocal。Spring 提供 TaskDecorator/DelegatingSecurityContextExecutor 等机制帮助把 MDC/安全上下文拷贝到异步线程。

> 这些绑定的“生命周期”都由**事务边界/请求边界**控制，不是“每个线程长期持有一个连接”。因此不会无节制占用连接池：**只有在当前线程参与事务/请求期间绑定，结束时释放**。



#### TransmittableThreadLocal

TTL 是对 InheritableThreadLocal 的增强：**在用线程池复用线程时，也能把“提交任务时”的上下文值传到“执行任务的线程”**；实现方式是**在提交时快照→执行前恢复→执行后还原**。 

> TTL 以“任务**创建/包装**那一刻”的父线程值为准（**快照语义**）。

```java
import com.alibaba.ttl.TransmittableThreadLocal;
import com.alibaba.ttl.TtlRunnable;
import com.alibaba.ttl.threadpool.TtlExecutors;
import java.util.concurrent.*;

public class DemoTTL {
    static final TransmittableThreadLocal<String> CTX = new TransmittableThreadLocal<>();

    public static void main(String[] args) throws Exception {
        ExecutorService raw = Executors.newFixedThreadPool(4);
        ExecutorService exec = TtlExecutors.getTtlExecutorService(raw); // 关键包装

        CTX.set("user:42");           // 提交前设置上下文（被快照）
        exec.submit(() -> System.out.println(CTX.get())).get(); // -> user:42

        // 也可只包装任务而非池：
        exec.submit(TtlRunnable.get(() -> use(CTX.get()))).get();
    }

    static void use(String v) { /* ... */ }
}
```

- TtlExecutors.getTtlExecutorService(...)：包装线程池，自动对提交的 Runnable/Callable 做快照/恢复。 
- 不包装线程池，也可用 TtlRunnable.get(...) / TtlCallable.get(...) 包装**每个**任务。 



在 CompletableFuture / 并行流 里传递上下文

CompletableFuture/并行 Stream 底层用 ForkJoinPool；TTL 已支持这一路径。最稳妥的方式：**显式提供已包装的执行器**。

```java
ExecutorService exec = TtlExecutors.getTtlExecutorService(new ForkJoinPool());

CTX.set("traceId=abc"); // 被快照
CompletableFuture
    .supplyAsync(() -> CTX.get(), exec)
    .thenApplyAsync(v -> v + "!", exec)
    .thenAcceptAsync(System.out::println, exec)   // -> traceId=abc!
    .join();
```



Spring 线程池 / @Async 中传播

```java
@Bean
public Executor asyncExecutor() {
    var t = new ThreadPoolTaskExecutor();
    t.setCorePoolSize(8);
    t.initialize();
    return com.alibaba.ttl.threadpool.TtlExecutors.getTtlExecutor(t.getThreadPoolExecutor());
}
```



### #3.14.1

#### 死锁

四要素：互斥 + 持有且等待 + 不可剥夺 + **循环等待**。工程里最常见的是**嵌套获取多个锁**时顺序不一致。



规避策略（从强到弱）

**统一加锁顺序（资源有全序）——首选**

```java
class Account {
    final int id; final Object lock = new Object();
    // ...
}
void transfer(Account a, Account b, int amt) {
    Account first  = a.id < b.id ? a : b;
    Account second = a.id < b.id ? b : a;
    synchronized (first.lock) {
        synchronized (second.lock) {
            // critical section
        }
    }
}
```

**tryLock(timeout) + 回滚/重试（打破“不可剥夺”）**

```java
boolean lockedA = lockA.tryLock(50, TimeUnit.MILLISECONDS);
if (!lockedA) return false;
try {
    if (!lockB.tryLock(50, TimeUnit.MILLISECONDS)) {
        // 回滚本线程已做的变更（若有），然后重试或上报
        return false;
    }
    try {
        // 临界区
    } finally { lockB.unlock(); }
} finally { lockA.unlock(); }
```

**降低锁粒度/拆分热点（Lock splitting/striping）**

比如 map 分段锁、队列读写分离，用 ReentrantReadWriteLock 或无锁/无阻塞结构（ConcurrentHashMap、LongAdder 等）。

**开放调用（Open call）**

**不要**在持锁期间进行**阻塞/外部调用**（RPC、I/O、日志重写、回调），先采数据→**释放锁**→再慢操作。



#### 活锁

线程没有阻塞，但不断“让步→重试”，系统吞吐趋近 0。最常见来源：**对称的冲突处理策略**（两个线程同时检测到冲突，同时放弃，同时再试…反复）。

修复方式：**随机退避**

```java
// 两线程都用 tryLock，拿不到就立刻让步，会活锁；加“抖动”可化解
Random rnd = new Random();

while (true) {
    if (lockA.tryLock()) {
        try {
            if (lockB.tryLock()) {
                try { /* work */ break; }
                finally { lockB.unlock(); }
            }
        } finally { lockA.unlock(); }
    }
    // 关键：退避且“随机化”，避免对称重合
    LockSupport.parkNanos(TimeUnit.MICROSECONDS.toNanos(50 + rnd.nextInt(200)));
}
```



#### 饥饿

常见来源

- **非公平锁** + 高频竞争：某些线程长期插队不上岸。

  - 解决：必要处使用公平锁（new ReentrantLock(true) / new Semaphore(permits, true) / new ReentrantReadWriteLock(true)），或降低锁时间。

  

- **读者-写者倾斜**：大量读锁让写锁长期饿死。

  - 解决：启用公平策略，或改为 StampedLock（写锁倾斜友好，注意其**不可重入**与 API 差异）。

  

- **线程优先级/OS 调度**：高优线程占满 CPU。

  - 解决：避免玩优先级；用默认优先级，靠并发结构限速。

  

- **线程池饥饿（经典）**：**固定大小线程池**内的任务在执行中**同步等待**同池提交的子任务 → 子任务永远得不到执行名额，形成“**线程池诱发的死锁/饥饿**”。

  - 复现（反例）：

  ```
  ExecutorService pool = Executors.newFixedThreadPool(1);
  pool.submit(() -> {
      Future<?> f = pool.submit(() -> {/* child */});
      f.get(); // 主任务阻塞等待“同池子任务” -> 永远卡住
      return null;
  }).get();
  ```

  - 修复：

    - 不在同池里**同步等待**子任务；用**非阻塞组合**（CompletableFuture 的 then* / allOf）。
    - 子任务改用**不同的执行器**；或增大池、按需线程（newCachedThreadPool），ForkJoin 场景用 ManagedBlocker（JDK）宣告阻塞。

    

  