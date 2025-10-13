# **一、CLH 是什么（本尊版本）**

**CLH（Craig–Landin–Hagersten）队列锁**是一个**FIFO 自旋队列锁**，线程按顺序在队尾排队；每个线程只“盯着”**前驱节点的状态**来决定自己是否可以获得锁，因此**局部自旋、缓存友好、可扩展**。



## **1) 节点结构（经典 CLH）**

```
// 伪代码
struct QNode { volatile bool locked; }  // true = 前驱持锁/未让位
```



## **2) 获取 / 释放（经典 CLH 模板）**

```
// 每个线程持有两个指针：myNode、myPred（可复用自己的节点）
lock() {
  myNode->locked = true;
  myPred = atomic_swap(&tail, myNode);      // 入队：把自己放到队尾
  while (myPred->locked) pause();           // 自旋观察“前驱是否放行”
}

unlock() {
  myNode->locked = false;                   // 放行给后继
  myNode = myPred;                          // 复用前驱节点，减少分配
}
```

**要点**

- **FIFO 公平**：顺序排队。
- **局部自旋**：只轮询前驱标志，缓存命中率高；与 MCS 一样属于“本地自旋队列锁”。
- **O(1) 远程访问**：每线程只有前驱一个远程共享点。

> MCS vs CLH：MCS 自旋在**自己的节点**上，前驱在释放时显式把“我的节点”置放行；CLH 自旋在**前驱节点**上，前驱仅清自己的标志即可。两者都扩展性好。





# **二、AQS 的 CLH 变体（Java 里天天见）**

JUC 的锁/同步器（ReentrantLock/ReadWriteLock/Semaphore/CountDownLatch …）都继承自 **AQS**。AQS 用的是**基于 CLH 思想的显式双向链队列**，把传统“自旋”换成了**挂起/唤醒（LockSupport.park/unpark）**，从而避免忙等烧 CPU。

## **1) AQS 节点（精简）**

```
static final class Node {
  // 模式
  static final Node EXCLUSIVE = null;
  static final Node SHARED = new Node();

  // 等待状态
  static final int CANCELLED =  1;   // 取消
  static final int SIGNAL    = -1;   // 前驱释放时要唤醒我
  static final int CONDITION = -2;   // 条件队列中
  static final int PROPAGATE = -3;   // 共享传播
  volatile int waitStatus;

  // CLH 链
  volatile Node prev;
  volatile Node next;
  volatile Thread thread;
  // ...
}
```

- waitStatus=SIGNAL(-1)：表示**前驱**在释放/传播时需要负责**唤醒我**。
- 头尾指针：head/(dummy) 和 tail 维护一个 FIFO。





## **2) 入队（enq）**

```
// CAS 把自己链接到 tail 后面；必要时创建 head 哨兵
Node enq(Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // 初始化
      if (compareAndSetHead(new Node())) tail = head;
    } else {
      node.prev = t;
      if (compareAndSetTail(t, node)) { t.next = node; return t; }
    }
  }
}
```







# **三、AQS 的 acquire/release 模板（独占）**

这是你在 ReentrantLock 等里看到的核心套路。

## **1) acquire（独占）**

```
public final void acquire(int arg) {
  if (tryAcquire(arg)) return;                  // 快路径：直接占到资源
  Node node = addWaiter(Node.EXCLUSIVE);        // 入队（CLH 尾部）
  boolean interrupted = false;
  for (;;) {
    Node p = node.predecessor();
    if (p == head && tryAcquire(arg)) {         // 轮到我了，再次尝试
      setHead(node); p.next = null;             // 我成为新 head（dummy）
      if (interrupted) selfInterrupt();
      return;                                   // 线性化点
    }
    // 还没轮到/失败：决定是否需要 park
    if (shouldParkAfterFailedAcquire(p, node))  // 把 p.waitStatus 置 SIGNAL
      LockSupport.park(this);                   // 挂起等待前驱唤醒
    if (Thread.interrupted()) interrupted = true;
  }
}
```

**关键点**

- **快路径**：先 tryAcquire（由具体同步器实现：如 ReentrantLock 检查可重入/占用）。
- **排队**：入队后只盯**前驱 p**（CLH 思想）。
- **唤醒链路**：shouldParkAfterFailedAcquire 会把 **前驱的 waitStatus 置为 SIGNAL**，表示“**当你释放时，记得唤醒我**”，然后我就 park 等。
- **省电**：不自旋烧 CPU，靠 park/unpark。





## **2) release（独占）**

```
public final boolean release(int arg) {
  if (tryRelease(arg)) {                     // 变更状态（如重入计数归零）
    Node h = head;
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);                    // 从 head 开始唤醒第一个有效后继
    return true;
  }
  return false;
}
```

- **释放快路径**：由具体同步器实现 tryRelease（如 ReentrantLock 的持有计数减到 0）。
- **唤醒**：若队列非空，**唤醒 head 的第一个合格后继**（跳过取消节点等）。
- **唤醒后继** = 把后继从 park 中叫醒，回到它的自旋处再次判断 p==head && tryAcquire，于是“**前驱→后继**”形成 CLH 风格的**传递**。







# **四、AQS 的 acquire/release 模板（共享）**

用于 Semaphore、CountDownLatch、ReentrantReadWriteLock.readLock() 等“**可共享**”的资源。

## **1) acquireShared**

```
public final void acquireShared(int arg) {
  if (tryAcquireShared(arg) < 0)           // <0 表示未拿到，需排队
    doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
  Node node = addWaiter(Node.SHARED);
  boolean interrupted = false;
  for (;;) {
    Node p = node.predecessor();
    if (p == head) {
      int r = tryAcquireShared(arg);       // 成功则返回 >= 0
      if (r >= 0) {
        setHeadAndPropagate(node, r);      // 我成为 head，并按需要“传播”唤醒更多共享后继
        if (interrupted) selfInterrupt();
        return;
      }
    }
    if (shouldParkAfterFailedAcquire(p, node))
      LockSupport.park(this);
    if (Thread.interrupted()) interrupted = true;
  }
}
```



## **2) releaseShared**

```
public final boolean releaseShared(int arg) {
  if (tryReleaseShared(arg)) {
    doReleaseShared();                     // 从 head 开始，按“共享传播”唤醒后继
    return true;
  }
  return false;
}
```

**要点**

- tryAcquireShared / tryReleaseShared 由具体同步器定义（如可用许可证的数量、读锁共享规则）。
- **传播**：共享模式可能一次唤醒不止一个后继（比如还剩配额），AQS 用 PROPAGATE/setHeadAndPropagate 协调“继续放行”。







# **五、把“CLH 思想”与 AQS 对齐**

| **经典 CLH**              | **AQS 变体（Java）**                                       |
| ------------------------- | ---------------------------------------------------------- |
| 节点：locked 布尔         | 节点：waitStatus（SIGNAL/CANCELLED/...）、prev/next/thread |
| 入队：swap(tail, myNode)  | enq() CAS 维护 tail，双向链                                |
| 自旋：while (pred.locked) | **不忙等**：shouldPark… → park，由前驱释放时 unpark        |
| 释放：myNode.locked=false | tryRelease() 成功 → unparkSuccessor(head)                  |
| 局部观察“前驱状态”        | 也是：节点只与**前驱/后继**交互，FIFO 传递                 |





# **六、你会用到的“自定义同步器模板”（扩展点）**

自己写一个 AQS 同步器，只需要实现 **两个钩子**（独占或共享）：

**独占型**

```
class MyMutex extends AbstractQueuedSynchronizer {
  protected boolean tryAcquire(int acquires) {
    // e.g. state 从 0->1 的 CAS，记录独占线程，支持可重入则检查持有者等
  }
  protected boolean tryRelease(int releases) {
    // e.g. state 从 1->0，清理独占线程
  }
}
```

**共享型**

```
class MySemaphore extends AbstractQueuedSynchronizer {
  protected int tryAcquireShared(int permits) {
    // e.g. 许可够就 CAS 扣减，返回 >=0；否则 <0
  }
  protected boolean tryReleaseShared(int permits) {
    // e.g. CAS 增加许可，返回 true 触发传播唤醒
  }
}
```

**其它模板**

- acquireInterruptibly/tryAcquireNanos：在上面循环里加**中断/超时**判定。
- ConditionObject：AQS 自带条件队列，await/signal 让节点在“条件队列”与“同步队列”之间转移（waitStatus=CONDITION）。







# **七、常见坑/要点**

- **waitStatus 只由前驱写**：SIGNAL 语义是“前驱负责唤醒我”，不要跨节点乱改。
- **取消节点清理**：CANCELLED(1) 的节点要在入队/唤醒路径上**跳过**，维持链表健康。
- **公平性**：AQS 默认非公平；公平锁会在 tryAcquire 里拒绝“插队”（检查 hasQueuedPredecessors()）。
- **避免忙等**：AQS 通过 park/unpark 替代自旋，是“阻塞式 CLH 队列”。
- **可见性**：所有关键字段均 volatile；CAS + LockSupport 保障 hb。







## **一页速记**

- **CLH**：FIFO 队列锁；**盯前驱**；释放=前驱清标志，后继通过。
- **AQS（CLH 变体）**：节点+队列；acquire：tryAcquire→入队→设置前驱SIGNAL→park；release：tryRelease→unparkSuccessor(head)；共享模式带**传播**。
- **自定义同步器**：只实现 tryAcquire/tryRelease（或共享版），其他排队与唤醒 AQS 替你搞定。



# 八、AQS 异常处理

```
Tcur: 取消自己
  node.waitStatus = CANCELLED
  try unlink:
    - CAS tail? or CAS pred.next?   // 可能失败
    - else unparkSuccessor(node)     // 把叫醒责任交出去

释放者: release
  unparkSuccessor(head)
    - 若 head.next 无效 → 从 tail 向前找第一个未取消节点 S 并 unpark(S)

S（被唤醒的后继）:
  while (pred.waitStatus > 0) pred = pred.prev; // 跳过取消的前驱
  （必要时 CAS 回填 pred.next = S）
  if (pred == head && tryAcquire()) 成功→ setHead(S)
  else → shouldPark... 再睡（先置 pred.SIGNAL，再 park）
```

## 为什么会失败（典型场景）

cancelAcquire(node) 的目标是：node.waitStatus=CANCELLED 并尽量让 pred.next 直接指向 node.next。但会遇到这些竞争场景：

1. **我是队尾（node == tail）**

   - 需要先把 tail CAS 回滚到 pred，再把 pred.next 置 null。
   - 任一 CAS 被并发修改抢走 ⇒ 失败，留给后续路径修复。

   

2. **我是 head.next（pred == head）**

   - 这个位置的“放行与唤醒”通常由**释放路径**负责（unparkSuccessor(head)），自己不强行改 head.next，否则容易和释放方打架 ⇒ 走兜底。

   

3. **我的后继不存在或也被取消（next == null 或 next.waitStatus > 0）**

   - 没有“合格后继”可以直接接上 ⇒ 不能就地重连，只能交给释放路径从队尾向前**找第一个未取消的后继**来唤醒。

   

4. **前驱状态还没准备好 / CAS 抢不过**

   - 规则要求“前驱负责唤醒我”，所以要把 **pred.waitStatus 设为 SIGNAL(-1)**。
   - 若 CAS 失败（并发竞争）或前驱已变化 ⇒ 放弃就地重连，改走 unparkSuccessor(node)。

   

5. **指针已被别人先修了**

   - 在你动手的同时，释放者或别的后继已经“帮你”跳过/修复了一部分指针 ⇒ 你的 CAS 自然失败，但这其实是**好事**（已经修好了）。





## **0) 关键背景与不变式**

**节点字段（关键）**

- prev / next / thread：双向链 + 拥有线程。
- waitStatus：SIGNAL(-1)=前驱释放需唤醒我；CANCELLED(1)=我放弃；PROPAGATE(-3)=共享传播；CONDITION(-2)=条件队列；0=无特殊状态。



**谁修改谁**

- **后继**设置**前驱**为 SIGNAL（“到时你要叫我起床”）。
- **释放者**叫醒**head 的有效后继**（或兜底扫描）。
- **取消者**把**自己**标成 CANCELLED，并**尝试**把自己从链表摘掉（不保证一次成功）。
- **被唤醒的后继**会**跳过取消的前驱**并回填必要指针。





**核心不变式**

1. head 是**哨兵**（获取者成功后把自己升为新的 head），真正等待队首应是 head.next。
2. waitStatus > 0 的节点视为**垃圾/取消**，行进中要跳过。
3. **不丢唤醒**：先将前驱置 SIGNAL，再 park；释放时清 head.waitStatus 并 unpark 有效后继。







## **1) 可中断获取被中断 / 超时 / 主动取消 ⇒**cancelAcquire(node)

**触发者**：当前线程（节点 node.thread 所属线程）。

**目标**：把自己标为取消，并尽力让队列继续前进（不丢唤醒，保持链路）。

**精简伪代码（贴近 JDK 实现）**

```
void cancelAcquire(Node node) {
  if (node == null) return;
  node.thread = null;

  // 1) 回溯跳过已取消前驱，接上最近未取消的前驱
  Node pred = node.prev;
  while (pred.waitStatus > 0) node.prev = pred = pred.prev;
  Node predNext = pred.next;

  // 2) 标记我为 CANCELLED
  node.waitStatus = CANCELLED;

  // 3) 三种尝试摘除/兜底路径
  if (node == tail && CAS(tail, node, pred)) {
    CAS(pred.next, predNext, null);                 // 我是尾：回滚 tail 并清 pred.next
  } else {
    int ws = pred.waitStatus;
    // 非头且让 pred 承诺 SIGNAL 成功，且我能看到有效后继：尝试 pred 直连我的后继
    if (pred != head &&
        (ws == SIGNAL || (ws <= 0 && CAS(pred.waitStatus, ws, SIGNAL))) &&
        pred.thread != null) {
      Node next = node.next;
      if (next != null && next.waitStatus <= 0)
        CAS(pred.next, predNext, next);             // 物理越过我
    } else {
      unparkSuccessor(node);                         // 兜底：把“唤醒责任”交出去
    }
  }

  // 4) 断开我的 next，帮助 GC/防误用
  node.next = node;
}
```

**会失败/走兜底的典型原因**

- 我是 **tail**，回滚 tail 或清 pred.next 的 CAS 输给并发。
- 我是 **head.next** 位置（交叉竞争敏感位），更稳妥由释放路径唤醒修复。
- 我没有可用后继（next == null 或也取消），无法就地重连。
- 正在竞争/指针已被别人先修，CAS 失败——这其实是**好事**（说明已经修过）。



**示意（我取消，尽力越过我）**

```
pred -> [ node(CANCELLED) ] -> succ
  |         ^   ^              ^
  |         |   |              |
  +---CAS---+   +----CAS------>+
         (pred.next = succ)     (或 unparkSuccessor(node))
```





## **2) 释放者** **release**⇒ **unparkSuccessor(head)**（含“从尾向前兜底扫描”）

**触发者**：成功释放资源的线程。

**目标**：叫醒**最近的有效后继**，哪怕 head.next 暂时失效，也要保证**队列前进**。

**精简伪代码**

```
void unparkSuccessor(Node h) {
  int ws = h.waitStatus;
  if (ws < 0) CAS(h.waitStatus, ws, 0);    // 清 SIGNAL/PROPAGATE

  Node s = h.next;
  if (s == null || s.waitStatus > 0) {     // next 无效：兜底路径
    s = tail;
    // 从尾向前，挑“离 head 最近”的有效节点（扫描到 head 即停）
    while (s != null && s != h && s.waitStatus > 0) {
      s = s.prev;
    }
  }
  if (s != null) LockSupport.unpark(s.thread);
}
```

**为什么“从尾向前”也能维护公平？**

被唤醒的 s 仍需满足 s.predecessor() == head && tryAcquire(...) 才能真正获得；否则它会把前驱置 SIGNAL 再睡。兜底扫描只在 head.next 失效时触发，唤醒**离 head 最近**的活节点，**最小修复成本**、保证进展。

**失效示意（X、Y 已取消；Z 是最近有效后继）**

```
head -> X(cancelled) -> Y(cancelled) -> Z(waiting) ; tail = Z

release:
  head.next 无效 → 从 tail 逆扫 → 选中 Z → unpark(Z)
Z 醒来:
  prev=Y(cancel)→X(cancel)→head
  现在 prev==head，tryAcquire 成功 → setHead(Z)
```





## **3) 被唤醒的后继在线程获取路径里的“自修复”**

**触发者**：被 unpark 的等待线程。

**目标**：**向前**跳过取消的前驱，把自己放到正确位置，并尝试获取。

**精简伪代码（独占模式热循环片段）**

```
for (;;) {
  Node p = node.predecessor();
  if (p.waitStatus > 0) {                      // 前驱是取消垃圾
    do { node.prev = p = p.prev; } while (p.waitStatus > 0);   // 向前跳过一串取消
    //（可选）CAS 回填 p.next = node
  }
  if (p == head && tryAcquire(arg)) {          // 真正轮到我
    setHead(node); p.next = null;              // 我成新 head（哨兵）
    return;                                    // 线性化点
  }
  if (shouldParkAfterFailedAcquire(p, node))   // 先让前驱承诺 SIGNAL
    LockSupport.park(this);                    // 再睡，不会丢唤醒
}
```

**要点**

- **不是“向后遍历”**，是**沿 prev 向前**清理取消的前驱。

- shouldParkAfterFailedAcquire 会：

  - 若 p.waitStatus == 0：CAS 置 SIGNAL；
  - 若 p.waitStatus > 0：跳过取消前驱；
  - 若 p.waitStatus == SIGNAL：可以安全 park。

  





## **4) 非可中断获取中的“中断”如何处理（延迟中断）**

**场景**：acquire(arg)（不可中断版本）中被中断。

**行为**：不立即 cancelAcquire，而是在循环里**记账**（interrupted=true），等**拿到锁后**调用 selfInterrupt() 把中断语义交还给上层。这保证队列不被无谓扰动。



## **5) 共享模式的“传播兜底”**

在共享获取/释放（如 Semaphore、读锁）中：

- tryAcquireShared(arg) 返回值 r >= 0 表示可以继续放行更多后继；

- setHeadAndPropagate(node, r) 会在设置新 head 后，根据 r 或 head.waitStatus==PROPAGATE 决定**继续唤醒**下一个共享后继。

  这同样遵循上面的“优先 head.next，失效则兜底扫描”的策略。





## **6) 一页“异常维护”流程速查（ASCII）**

**A. 取消/中断/超时（当前线程）**

```
[... -> pred ] -> node -> [ succ ... ]
             ↑            ↑
         (valid)      (my thread)

1) node.waitStatus = CANCELLED
2) 尝试:
   - 我是 tail? CAS(tail=pred); CAS(pred.next=null)
   - 否则 pred != head 并承诺 SIGNAL 且 succ 有效? CAS(pred.next=succ)
   - 否则 unparkSuccessor(node)
3) node.next = node   (断开)
```

**B. 释放兜底（head.next 失效）**

```
head -> X(cancelled) -> Y(cancelled) -> Z(waiting) ... tail=Z

release:
  s = head.next 无效 → s = tail; while (s!=null && s!=head && s.cancelled) s=s.prev;
  unpark(s)   // s 会沿 prev 跳过取消前驱，贴近 head 再竞争
```

**C. 被唤醒后继自修复**

```
Z 被唤醒:
  p=Y(cancel) → p=X(cancel) → p=head
  p==head && tryAcquire ?  成功→ setHead(Z) : (让 head 承诺 SIGNAL 再 park)
```





## **7) 你可以记住的 5 条“铁律”**

1. **取消一定先标记自己**：waitStatus=CANCELLED，并**尝试**摘除自己；不成功也会有**兜底**。
2. **不丢唤醒**：先把前驱置 SIGNAL，再 park；释放时从 head 开始唤醒（必要时兜底扫描）。
3. **兜底扫描**只在 head.next 失效时发生，且唤醒**离 head 最近**的活节点，保证进展。
4. **后继醒来只“向前跳”**：跳过取消前驱、回填必要指针，再判断 p==head 竞争。
5. **Node 不“自执行”**：所有维护由 AQS 模板在不同参与者（取消者/释放者/后继）路径上“协作完成”。



## 总结（重点）

```
初始（a,b,c 已取消）：
head -> a(X) -> b(X) -> c(X) -> d(?) -> e(?) -> f(?)     ; tail = f
```

**步骤 1：head 释放 →** **unparkSuccessor(head)**

- 发现 head.next = a，且 a.waitStatus > 0（取消） ⇒ 进入兜底扫描。

- **真实代码**（JDK AQS）逻辑是：从 tail 一路往前扫到 head，把遇到的“未取消（waitStatus <= 0）”的节点依次赋给变量 s，**最终留下的 s 就是“离 head 最近的未取消节点”**。

  在例子里：经历 f → e → d，停在 d 上，所以**唤醒的是 d**（不是 f）。



**步骤 2：d 被唤醒后的获取循环**

```
d 醒来：
  p = d.prev = c(X)  → 跳过  → b(X) → 跳过 → a(X) → 跳过 → p = head
  现在 p == head → tryAcquire(...) 成功 → setHead(d)；旧 head.next 置 null
```



结果：

```
head(=d) -> e -> f     // a,b,c 已被“跳过/断开”，队列恢复健康
```

> 关键：**“跳过 a,b,c”** 发生在 **d 醒来后** 的获取循环中——它沿 prev 向前跳过取消前驱，并可顺带回填前向指针；**不是**“先唤醒 f→f 再唤醒 e→e 再唤醒 d”的级联。

## 为什么 AQS 会选择从后往前而不是从前往后找最近一个未取消的节点

答案的关键在于 **AQS 的“入队与链接时序”**：

**prev 链是“强一致”的，next 链是“最佳努力、可能暂时断开的”。**

这就是为什么 unparkSuccessor(head) 在发现 head.next 无效时，会**从 tail 沿 prev 向前回溯**，而不是从 head 沿 next 向前找。

**1) 入队时序决定了“prev 可靠、next 可能缺口”**

入队（enq）的关键步骤（简化）：

1. node.prev = tailPred;    ← **先写 prev（立刻可靠）**
2. CAS(tail, tailPred, node);  ← **把 tail 指到新节点**
3. tailPred.next = node;    ← **最后才补上前驱的 next**



在第 2 步成功到第 3 步生效之间存在一个**窗口**：

- tail 已经指向 **node**；
- 但 **tailPred.next 还没来得及写** → **next 链出现“缺口”**。



**ASCII：入队“next 缺口”**

```
... -> tailPred        tail = node
          \
           (step 3 未完成，next 还没连上)
            \
head -- ...   node(prev=tailPred)  next=null (暂时)
```

**结论**：

- **沿 prev 从 tail 回溯**，总能走到 head，**不会漏**；
- **沿 next 从 head 前行**，**可能遇到 null 就“误以为队列空”**（其实后面还有节点）。



**2) 取消/并发修复会进一步“扰乱 next”，但不影响** **prev**

- 取消时，节点会被标记 CANCELLED，并尝试 pred.next = succ；有时还会把自己的 next 设为自指来防误用。
- 多线程并发“清垃圾”时，**next 指针可能暂时不完整**或被别的线程刚刚重连；
- 但 **prev 链是入队时就确定/可传递的**（每个节点自己写自己的 prev），**始终可靠**。

> AQS 的很多修复/兜底逻辑都遵循一个原则：**“以 prev 链为真相，以 next 链为可选回填。”**



**3)** **unparkSuccessor(head)**的目标与策略

目标：在 head.next 无效（null 或已取消）时，**唤醒“离 head 最近的未取消节点”**，保证队列前进且尽量公平。

策略（伪代码含意）：

- 先看 head.next；可用就直接唤醒。
- 否则 s = tail，**沿 prev 回溯**直到 head，一路**跳过取消节点**；保留“最后一个 waitStatus <= 0 的节点”作为候选 **s**；
- **s 就是“离 head 最近的活节点”** → unpark(s.thread)。



**ASCII：**head.next失效下的兜底

```
head -> a(X) -> b(X) -> c(X) -> d(ok) -> e(ok) -> f(ok)   tail=f

从 tail 回溯： f(X?)→e(ok)→d(ok)→c(X)→b(X)→a(X)→head
最终候选 = d   （离 head 最近的活节点）
```

**为什么不从前往后？**

因为此时 head.next 可能因为“入队 next 缺口”或“并发清理”而是 null/垃圾，**向前走会误判**。**从尾回溯沿 prev 则永远不会漏**，还能一次性跳过一串取消节点，选中**最近**的活节点。



**4) 公平性 & 进展性**

- **公平性**：即使唤醒的是从尾回溯得到的节点，被唤醒线程依然要满足 p == head && tryAcquire(...) 才能获取；否则会把 pred.waitStatus 置为 SIGNAL 再次休眠。最终能成功的那个，**确实是“逻辑上的 head.next”**。
- **进展性**：当 head.next 失效、next 链有空洞时，如果不回溯，**可能没人被唤醒**而卡住；从尾回溯能**保证总有活节点被叫起**来帮忙修复并推进队列。



**5) 总结成 3 句**

1. **入队时 prev 先立、tail 先指、next 后补** ⇒ prev 链**可靠**、next 链**可能暂时断**。
2. unparkSuccessor(head) 在 head.next 无效时，**从 tail 沿 prev 回溯**，挑“**离 head 最近**的未取消节点”唤醒，**既不漏也不乱**。
3. 这是 AQS 为了**避免漏唤醒**、**保证进展**且**维持近似公平**的工程取舍；“从前往后”在临时断链情况下会误判队列空，风险更大。

这样再看 AQS 的代码就顺了：**prev 是权威链，next 是回填链**；在异常（取消/断链）时，靠 prev 才能“稳准快”地找到应唤醒的那个等待者。