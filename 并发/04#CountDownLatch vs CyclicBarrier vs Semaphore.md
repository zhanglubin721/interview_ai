# **CountDownLatch**

## **能干什么（一句话）**

**一次性闩锁**：等待 **N 个事件**（或线程）完成后，**一次性**打开闩锁，所有等待者同时通过；**不可复用**。

## **内部实现（AQS 共享模式）**

- **状态值 state = count**（构造时确定，之后**只减不增**）。
- await()：acquireSharedInterruptibly(1) → tryAcquireShared 若 state==0 返回 1（通过），否则返回 -1 进入同步队列等待。
- countDown()：CAS 循环把 state 减 1；当减到 0 时调用 releaseShared(1)，**唤醒所有在同步队列里的等待者**。
- **不可重置**：没有再把 state 设回去的 API（这就是 “一次性” 的根源）。
- **可见性**：基于 AQS 的 park/unpark 与 volatile/CAS，countDown 之前的写在 await 返回后可见（hb 链）。



## **典型用法（业务）**

- **主线程等子任务**：拆分 N 个任务并行处理，主线程 await()，子任务完成 countDown()。
- **“开始枪”与“结束枪”**（两把闩）：一个 latch 让所有工作线程同时开始；另一个 latch 等所有工作线程结束。
- **服务就绪**：主流程等数据库/缓存/消息系统 N 个依赖组件完成初始化。
- **压测**：同时起跑，减少干扰。

> 小提示：countDown() 写在 finally；多个等待者都可 await()；**任何线程**都能 countDown()（不要求是谁启动的）。



# **CyclicBarrier**

## **能干什么（一句话）**

**可复用的栅栏**：**N 个“参与方”来到同一栅栏点时，一起放行进入下一代（generation）**；可选一个 **barrierAction** 在每次放行时**由最后到达者**执行。

## **内部实现（Lock + Condition，不走 AQS）**

- 字段：parties（总人数）、count（还差几人）、generation（一代栅栏的标识）、broken（破栏标志）、可选 barrierCommand。

- await() 流程（简化）：

  1. lock.lock()；int index = --count。

  2. 若 index == 0（**最后到达**）：

     - 先执行 barrierCommand（若有，**在持锁下**由最后到达者执行）；
     - nextGeneration()：重置 count=parties，生成新 generation，trip.signalAll() 唤醒所有在本代等待的人。

     

  3. 否则：while (generation 未变 && !broken) trip.await()（带中断/超时版本会抛出异常）。

- **破栏（broken）**：如果任一等待者 **中断** 或 **超时**，会把当前代标记为 broken=true，并 signalAll() 让大家都抛 BrokenBarrierException 返回，避免死等。

- **复用**：每次放行进入**下一代**（generation 对象更换），可 reset() 手动复位破栏。



## **典型用法（业务）**

- **迭代式并行算法**：每轮计算后等全体完成再进入下一轮（如并行模拟、数值迭代、图算法多轮收敛）。
- **阶段性流水线**：阶段 A 的多线程全到齐后，一起进入阶段 B。
- **对齐批次**：多个采集器对齐批次时间点一起刷写；barrierAction 做“收尾汇总”。

> 小提示：任意一个等待者**中断或超时**都可能把这一代**破栏**；使用时要设计好超时与兜底逻辑。

# **Semaphore**

## **能干什么（一句话）**

**计数信号量**：限制**同时访问一个资源的并发度**（permits 个许可）。permits=1 可充当**互斥量**；支持**公平/非公平**。



## **内部实现（AQS 共享模式）**

- **状态 state = permits**。
- acquire(n)：acquireSharedInterruptibly(n) → tryAcquireShared(n)
  - 读当前 state，若 state - n < 0 返回负数 → 入队等待；
  - 否则 CAS 把 state 减 n 成功 → 通过（非公平模式不检查排队，允许插队；公平模式先 hasQueuedPredecessors()）。
- release(n)：releaseShared(n) → CAS 把 state 加 n → 唤醒等待者（可能一次唤醒多个共享等待）。
- **不关心“持有者是谁”**：A 线程 acquire 后，B 线程也可 release（与锁不同）。



## **典型用法（业务）**

- **限流/并发度控制**：接口限 100 并发、线程池外的本地保护。
- **连接/句柄池**：控制同时打开的连接数量；availablePermits() 做监控。
- **一次性大额许可**：tryAcquire(n, timeout) 当成“批量借用资源”，不足则等待/失败。
- **互斥（permits=1）**：需要“可中断/可超时/可公平”的互斥时替代 ReentrantLock（但注意没有可重入性）。

> 小提示：drainPermits() 一次性取空（做维护）；reducePermits(k) 动态收紧上限；公平信号量避免饥饿但吞吐略降。





# **什么时候用谁？（快速决策）**



| **诉求**                                         | **推荐**                                |
| ------------------------------------------------ | --------------------------------------- |
| 等 N 件事完成后再继续，**一次性**放行            | **CountDownLatch**                      |
| N 个参与者在多个阶段**反复对齐**、一起进入下一轮 | **CyclicBarrier**（可加 barrierAction） |
| 控制**同时访问**的并发数/资源数；可当互斥或限流  | **Semaphore**                           |



# **三者内部核心对比（实现侧）**

| **组件**       | **核心实现**                  | **状态值**                  | **是否可复用** | **等待者在哪里**      |
| -------------- | ----------------------------- | --------------------------- | -------------- | --------------------- |
| CountDownLatch | **AQS 共享**                  | state = count（只减）       | ❌ 一次性       | AQS 同步队列（await） |
| CyclicBarrier  | **ReentrantLock + Condition** | count、generation、broken   | ✅ 多代复用     | 条件队列 trip         |
| Semaphore      | **AQS 共享**（公平/非公平）   | state = permits（可加可减） | ✅              | AQS 同步队列（共享）  |





# **小示例（各 10 行）**

## **CountDownLatch：主线程等 3 个子任务**

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
  new Thread(() -> {
    try { /* do work */ } finally { latch.countDown(); }
  }).start();
}
latch.await(); // 全部完成后继续
```



## **CyclicBarrier：3 个工作者每轮对齐，最后到达者汇总**

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("round done"));
for (int i = 0; i < 3; i++) {
  new Thread(() -> {
    for (int r = 0; r < 5; r++) {
      /* step r work */
      try { barrier.await(); } catch (Exception e) { return; } // 中断/超时会破栏
    }
  }).start();
}
```

**效果：**

- 每一轮（r = 0..4），3 个线程都跑到 barrier.await()；当**第 3 个（最后到达）的线程**到达时：
  1. 先执行 barrierAction → 打印一行 round done
  2. 然后同时放行本轮阻塞的 3 个线程进入下一轮

因此，整个程序会**打印 5 行**：

```
round done
round done
round done
round done
round done
```

每行对应一轮，打印者是**该轮最后到达的那个线程**（谁最后到达不可预测）。

**时序要点：**

- barrierAction 在**放行之前**执行，且**只执行一次/每轮**。
- 下一轮从 count 重置为 3 开始，循环往复 5 次。
- 主线程没 join()，但 3 个子线程是非守护线程，都会正常跑完后进程再退出。

**异常/中断（这段代码里被 catch 到会 return）：**

- 若某线程在 await() 时**中断/超时/动作抛异常**，屏障会被标记 **broken**，本轮及后续在这个 CyclicBarrier 上 await() 的线程都会收到 BrokenBarrierException，被 catch 后退出。正常情况下（不打断也不超时）就如上打印 5 行。



## **Semaphore：限制同时只允许 10 个任务执行**

```java
Semaphore sem = new Semaphore(10, true); // 公平可选
Runnable job = () -> {
  try {
    sem.acquire();
    /* use limited resource */
  } catch (InterruptedException e) {
    Thread.currentThread().interrupt();
  } finally {
    sem.release();
  }
};
```

# **业务选型与注意事项**

- **Latch**：一次性闩锁；countDown() 必须放 finally；getCount() 仅用于监控不做逻辑判断。
- **Barrier**：任何一人**中断/超时**会**破栏**，所有同代等待者抛 BrokenBarrierException；对齐人数必须正确；barrierAction 可能抛异常也会破栏。
- **Semaphore**：不是“谁拿的谁还”的强约束（可能跨线程释放）；**不具备可重入性**；公平模式防饥饿但吞吐更低；监控可用 availablePermits()。



**一句话收束**：

- **CountDownLatch**：一次性“等 N 件事”。

- **CyclicBarrier**：可复用“集齐 N 人一起走下一步”。

- **Semaphore**：限并发/限资源的“发许可证”。

  理解它们背后：**Latch/Semaphore 基于 AQS（共享），Barrier 基于 Lock+Condition**，再去看源码就顺了。