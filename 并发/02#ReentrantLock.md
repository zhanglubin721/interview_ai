# **一、背景（共同点）**

- 状态位：state（持有次数），exclusiveOwnerThread（持有者）。
- 队列：AQS 的 **同步队列**（CLH 变体，head 哨兵、tail 队尾；节点 Node{prev,next,thread,waitStatus}）。
- 唤醒规则：释放时从 head 开始唤醒**最近的有效后继**（unparkSuccessor(head)，必要时“从尾向前兜底扫描”）。

# 二、非公平锁（默认）——“两次抢占，不成再排队”

> 关键词：**允许插队（barging）**。即使队列里有人，新线程也可能直接拿到锁。



## **入口：**lock()

1. **快速 CAS**：CAS(state, 0→1) 成功 → setOwner(current) → **直接返回**。
2. 失败 → 走 acquire(1)（进入 AQS 模板）。



## **AQS 获取模板：**acquire(1)

1. **第 2 次即时抢占**：调用 tryAcquire(1)（非公平实现里是 nonfairTryAcquire）

   - state==0：**不看队列**，直接 CAS(0→1)，成功就返回（这一步就是 **barging**）。
   - state>0 且 **可重入**（owner==current）：state+=1，返回。
   - 否则失败 → 进入排队。

2. **入队**：addWaiter(EXCLUSIVE) → enq() CAS 尾插 → 得到自己的 node。

3. **阻塞循环**：acquireQueued(node, 1)

   - 取前驱 p = node.predecessor()；

   - **轮到我？** 如果 p==head 且 tryAcquire(1) 成功 → setHead(node) → 返回；

   - **未轮到**：shouldParkAfterFailedAcquire(p, node)

     - 把 **前驱 p.waitStatus 置为 SIGNAL**（承诺“释放时叫醒我”）
     - 然后 park() 挂起；被唤醒后回到循环第 1 行重试。

     

> 串起来：

> lock() 快速 CAS（抢占 1）→ 失败 → acquire(1) → tryAcquire（抢占 2）→ 失败 → 入队 → park 等唤醒 → 被叫醒后如果 p==head 再 tryAcquire 成功 → 晋升 head。



## **释放：unlock()**→ **release(1)**

1. tryRelease(1)：state-=1；若变 0 → 清 owner 并返回 true。

2. unparkSuccessor(head)：

   - 优先 head.next；若无效（null/取消）→ **从 tail 沿 prev 回溯**找**离 head 最近**的有效节点 s，unpark(s.thread)。

   



# **三、公平锁——“**前面有人就排队”

> 关键词：**不允许插队**。近似 FIFO。



## 入口：lock()

1. **没有快速 CAS**：直接 acquire(1)（不在 lock() 里抢）。



## **AQS 获取模板：**acquire(1)

1. 调 tryAcquire(1)（**公平实现**）

   - state==0：先查 hasQueuedPredecessors()
     - **有人在我前面** → 返回 false（我必须排队）
     - **没人** → CAS(0→1) 拿锁成功（**不入队**）
   - state>0 且可重入 → state+=1 成功返回
   - 否则失败 → 排队

   

2. **入队/阻塞**：与非公平一致（步骤 4—5）。

   公平的差别在于：即使我被唤醒，只要前驱不是 head，我也不会通过 tryAcquire 成功（因为前面还有人），会继续把前驱置 SIGNAL 再 park。



## **释放：与非公平一致（步骤 6—7）**



# **四、三条典型时序（ASCII）**

### **1) 非公平、队列已有等待者，新线程“插队”成功**

```
T1：入队等待    head -> [T1] ... tail
T2：lock()
  ├─ lock() 快速 CAS 失败
  ├─ acquire(1) → tryAcquire(非公平) 直接 CAS 成功（不看前驱）
  └─ T2 获锁；T1 继续在队首等
```



### **2) 公平、队列已有等待者，新线程“老老实实排队”**

```
T1：入队等待    head -> [T1] ... tail
T2：lock()
  ├─ 直接 acquire(1)
  ├─ tryAcquire(公平)：hasQueuedPredecessors() == true → 失败
  ├─ addWaiter → enq 入队
  └─ T2 挂起等待；释放后按照 FIFO 依次唤醒
```



### 3) 释放时head.next失效的兜底（两种锁都一样）

```
head -> a(cancelled) -> b(cancelled) -> c(waiting) ; tail=c

release:
  head.next 无效 → 从 tail 回溯：c 是离 head 最近的活节点 → unpark(c)
c 醒：沿 prev 跳过 a,b → p==head → tryAcquire 成功 → setHead(c)
```



# **五、把“容易误解”的几个点一次讲透**

1. **“非公平只抢一次？”** 否。**两次**：lock() 里一次、acquire 里 tryAcquire 再一次；都失败才入队。
2. **公平锁会不会也不入队直接拿？** 会。**前面没人**（hasQueuedPredecessors()==false）时，tryAcquire 直接 CAS 成功。
3. **tryLock() 的坑**：无参 tryLock() 在“公平锁”上**也可能非公平地直接抢**（通常直接调用非公平 nonfairTryAcquire）；若要在公平语义下等，请用 lock() / lockInterruptibly() / tryLock(timeout)。
4. **被唤醒后为什么还可能睡回去？** 因为**伪唤醒或竞争失败**；模板会先把前驱置 SIGNAL 再 park，不丢唤醒。
5. **为什么释放时从尾向前找？** 因为 prev 链可靠、next 链可能暂时断（入队“先 prev、后 next”的时序），尾向前能**保证找到离 head 最近的活节点**，队列不至于卡住。



# **六、一句话对比（面试速记）**

- **非公平**：lock() 先 CAS 抢 → 失败进 acquire 再 tryAcquire 抢 → 再失败才入队；**允许插队**，吞吐高。
- **公平**：lock() 直接 acquire；tryAcquire 先看 hasQueuedPredecessors()，**前面有人就排队**；近似 FIFO，吞吐略低。