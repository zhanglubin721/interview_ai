# **先导：happens-before 是什么**

- **定义**：若 A happens-before B，则 **A 的所有效果（写入）对 B 可见**，且 **A 在 B 之前按顺序发生**。这是 JMM 给编译器/JIT/CPU 设的“栅栏级顺序约束”，用来抵抗重排序与缓存不可见。
- **不是**：happens-before 并不等于“物理时钟更早”或“单核执行顺序”；它是一个 **偏序关系**，由若干规则连成边，再靠**传递性**闭包起来。
- **目标**：只要程序是“**正确同步**”（无数据竞争），JMM承诺**顺序一致性**的效果。



# **1）程序顺序规则（Program Order Rule）**

**同一线程内**，按照 **程序次序**，前面的动作 *happens-before* 后面的动作。

> 这是线程内的“顺序语义”，编译器仍可做不观测差异的重排，但不能破坏同线程 hb。

**能保证**

- 单线程视角里，你写在前、读在后，在这个线程内就按前后逻辑可见。

**不能保证**

- 对**其他线程**的可见性与顺序；线程间还需“同步边”。

**例子**

```
int a = 0, b = 0;

void t1() {          // 线程1
    a = 1;           // A1
    b = 2;           // A2  (A1 happens-before A2 in t1)
}

void t2() {          // 线程2
    // 不能仅凭“程序顺序”指望看到 a==1,b==2
    System.out.println(a + "," + b);
}
```



# **2）监视器锁规则（Monitor Lock Rule）**

对同一把监视器（同一个 synchronized 监视对象）：

- **解锁（unlock）** *happens-before* **后续**的加锁（lock）。



**等价直觉**

- 释放锁 = **写入释放（release）屏障**；获取锁 = **读取获取（acquire）屏障**。
- 释放前对共享内存的写，获取后都可见。



**例子**

```java
final Object LOCK = new Object();
int x = 0;

// Writer
synchronized (LOCK) {  // acquire
    x = 42;
}                      // release

// Reader（之后某时刻）
synchronized (LOCK) {  // acquire
    // 一定能看到 x == 42
    System.out.println(x);
}                      // release
```

**常见坑**

- **必须是同一把锁**。换对象、不同锁，不建立 hb。
- wait/notify：
  - notify/notifyAll **到** 被唤醒线程 wait **返回**之间也有 hb；
  - 但**条件**仍需在同一锁保护下反复检查（while 轮询条件），否则易丢信号/虚假唤醒。



# **3）**volatile**变量规则（Volatile Rule）**

对同一个 volatile 变量 v：

- 对 v 的**写** *happens-before* 之后对 v 的**读**。
- JMM 还要求：每个 volatile 变量的读/写形成一个与程序序一致的**全序**。



**等价直觉**

- volatile 写 = **release**；volatile 读 = **acquire**。
- 读到某次 volatile 写后，该写前的所有普通写也变得可见（靠**传递性**，见下节）。



**能保证**

- **可见性**与**有序性**（跨线程：写前的普通写对读后可见）。
- **不保证**复合操作的**原子性**（比如 count++ 仍然竞态；需 Atomic* 或锁）。



**示例：可见性开关**

```java
class Worker {
    // 若去掉 volatile，t2 可能永远看不到 flag == true
    private volatile boolean flag = false;
    private int data = 0;

    void t1() {            // 线程1
        data = 42;         // 普通写 W1
        flag = true;       // volatile 写 Vw（release）
    }
    void t2() {            // 线程2
        while (!flag) {}   // volatile 读 Vr（acquire）
        // 由于 Vw happens-before Vr，且有传递性 => 这里一定看到 data == 42
        System.out.println(data);
    }
}
```

**常见坑**

- volatile **不**提供“读-改-写”的原子性；++、putIfAbsent 等需 Atomic* 或锁。
- 不同的 volatile 变量之间**没有**额外的全局顺序（分别各自全序）。



# **4）传递性（Transitivity）**

若 **A hb B** 且 **B hb C**，则 **A hb C**。

这是把“释放→获取”边串起来的关键。上节示例中：

- data=42 **程序序**→ flag=true (volatile 写)
- flag=true (写) **volatile 规则**→ while(!flag) ... 处的 **volatile 读**
- **传递性**：data=42 对读线程可见。



# **5）线程启动 / 终止 / 中断（Start / Termination / Interrupt）**



## **5.1 启动（Thread.start）**

- 线程 A 中对 t.start() 的**调用** *happens-before* 线程 t 中的**任意动作**。

```java
int x = 0;
Thread t = new Thread(() -> {
    // 一定能看到 x==1
    System.out.println(x);
});
x = 1;        // 在 start 之前
t.start();    // A: happens-before t 里的 run 动作
```



## **5.2 终止（Thread.join）**

- 线程 t **内的所有动作** *happens-before* 另一个线程 **成功返回 t.join()** 之后的动作。

```java
int r;
Thread t = new Thread(() -> { r = compute(); });
t.start();
t.join();         // t 的写入对 join 之后可见
System.out.println(r);
```

> 要点：**必须**是“成功返回的 join()”这一同步点；单纯轮询共享变量、或只调用 isAlive() 不构成 hb（除非实现层另有保证，规范上以 join 为准）。



## **5.3 中断（Thread.interrupt）**

- 对 t.interrupt() 的**调用** *happens-before* 线程 t **检测到中断**（抛 InterruptedException，或 Thread.interrupted() / t.isInterrupted() 返回 true）。

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(10_000);
    } catch (InterruptedException e) {
        // interrupt() 的调用 happens-before 这里捕获到异常
    }
});
t.start();
t.interrupt();
```



# **小结速查（规则到效应）**

- **同线程程序序**：单线程前→后。
- **锁（unlock → lock）**：释放前的写，对之后持有同一锁的读可见。
- **volatile（write → read）**：写前的所有写，对读后可见（acquire/release 语义）。
- **线程启动**：start() 之前 → 子线程动作。
- **线程终止**：子线程所有动作 → 另线程 join() 返回之后。
- **线程中断**：interrupt() → 被中断方检测到中断。
- **传递性**：把上面任意边**串起来**形成更长的可见性链。



# **典型范式与对比**

## **1. 用** **volatile****做发布（轻同步，非复合原子）**

```java
class Holder { final int v; Holder(int v){ this.v=v; } }
volatile Holder h;

void writer() {          // 线程1
    Holder tmp = new Holder(42); // final 字段在构造内冻结
    h = tmp;                      // volatile 写，发布
}
void reader() {          // 线程2
    Holder local = h;            // volatile 读，获取
    if (local != null) use(local.v); // 一定看到 v == 42
}
```



## **2. 双重检查锁（DCL）必须配合** **volatile**

```java
class LazySingleton {
    private static volatile LazySingleton INSTANCE;
    private LazySingleton(){}

    static LazySingleton get() {
        if (INSTANCE == null) {              // 第一次检查（无锁）
            synchronized (LazySingleton.class) {
                if (INSTANCE == null) {      // 第二次检查（有锁）
                    INSTANCE = new LazySingleton(); // 需要 volatile 防止重排/可见性问题
                }
            }
        }
        return INSTANCE;
    }
}
```



## **3. 条件等待（**wait/notify**必须在同一锁 + 循环判条件）**

```java
final Object LOCK = new Object();
boolean ready = false;

void producer() {
    synchronized (LOCK) {
        // 写共享状态
        ready = true;
        LOCK.notifyAll();  // notify happens-before 消费者 wait 返回
    } // unlock → 可见
}

void consumer() throws InterruptedException {
    synchronized (LOCK) {
        while (!ready) {   // 必须 while，防虚假唤醒/丢信号
            LOCK.wait();   // 释放锁并挂起，醒来后会重新 acquire
        }
        // 这里一定能看到 ready == true
    }
}
```



# **常见误解澄清**

1. **“我读到某个 volatile 值，就能看到全局所有写”**：不对。**只**保证看到该 volatile 写**之前**的写（并靠**传递性**传播）。与之无关的并发写若没有被这条链覆盖，仍可能不可见。
2. **“有了 volatile，就不需要原子类/锁了”**：不对。volatile 不提供复合操作的原子性；++、putIfAbsent、计数器累加等要用 AtomicLong、LongAdder 或锁。
3. **“isAlive()/轮询字段等价于 join()”**：不对。**规范层**只有 join() 的 hb 保证能把被 join 线程内的写映射成可见。
4. **“不同锁之间能传可见性”**：只有**同一把锁的 unlock→lock** 才是 hb 边；不同锁之间需要另外的同步手段（例如 volatile、park/unpark、Future.get() 等）。

# **扩展（你会经常用到的 hb 边）**

- **类初始化**：类的初始化完成 *happens-before* 任何使用该类（读取静态字段、创建实例）。

- **LockSupport.unpark(t) → t.park() 返回**：unpark *happens-before* 被唤醒的 park 返回。

- **Executor/Future（J.U.C 文档的内存一致性效果）**：

  - 提交任务前的动作 *happens-before* 任务开始；
  - 任务内的动作 *happens-before* Future.get() 返回。

  

- **final 字段语义**：构造函数内对 final 字段的写，对**正确发布**后其他线程可见（防止“半初始化”重排）。

