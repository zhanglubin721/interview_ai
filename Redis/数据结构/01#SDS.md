# **1. SDS 是什么 & 设计目标**

SDS 是 Redis 在 C 里实现的一套“**二进制安全的可变字符串**”。它替代原生 char* 的原因主要有：

- **O(1) 取长度**：不用 strlen 线性扫描。
- **二进制安全**：内容里可以包含 '\0'。
- **自动扩容**：避免缓冲区溢出；追加摊还 O(1)。
- **与 C 生态兼容**：尾部依然放一个 '\0'，可当 C 字符串用（只读场景）。





# **2. 内存布局与头部结构**

SDS 在堆上长这样（指针 s 返回给你的是 buf 的起始地址）：

```
|<-----------  可  变  的  头  部  ----------->|<- 数据区 ->|'\0'|
+---------------------+------------------------+-----------+----+
|   len   |  alloc   |  flags  |     buf[]    |  ...data  | \0 |
+---------------------+------------------------+-----------+----+
^                                               ^
header 起点                                     s（即返回给用户的指针，指向 buf[0]）
```

- len：当前已使用长度（不含末尾 \0）。
- alloc：buf 可用总容量（不含 \0）。
- flags：低 3 位存储 **类型**（决定头部大小），其余位给某些紧凑变体使用。
- buf[]：弹性数组，真实数据，从 s 开始访问。

为节省小字符串的头部开销，SDS 有多种头部变体（都用了 __attribute__((packed)) 减少对齐填充）：

- sdshdr8 ：len/alloc 用 uint8_t，最大 255；
- sdshdr16 ：uint16_t；
- sdshdr32 ：uint32_t（常见，最大约 4GB）；
- sdshdr64 ：uint64_t（极大字符串）。
- 有的版本还保留了 sdshdr5（把长度的 5 bit 塞在 flags 里，支持 ≤31 字节的极短串）。是否启用取决于编译宏，在新版本/主流构建里常见做法是使用 8/16/32/64 这四档。

**关键点**：SDS 返回给你的类型是 typedef char* sds;，也就是说你拿到的是 buf 的指针。需要读写头部时，SDS 通过 s[-1] 先取 flags 判断类型，再用指针回跳到对应的头部结构（指针算术 + offsetof）来访问 len/alloc。

**不变式**：始终保证 buf[len] == '\0'，因此可以把它传给只读的 C API；但**不能**用可能越界写入的 C API 去改它（比如没扩容就 strcat）。





# **3. 为什么比** **char **更可靠

- **O(1) 长度**：sdslen(s) 直接读头部；C 的 strlen 必须扫到 \0。
- **二进制安全**：长度由 len 决定，不靠 \0 分隔，内容里放 \0 没问题。
- **自动扩容**：sdsMakeRoomFor 按策略增长，避免手写 realloc 出错。
- **可控的冗余**：用 “预分配 + 惰性回收” 减少频繁 realloc。





# **4. 容量管理策略（扩容、预分配、收缩）**

## **扩容（**sdsMakeRoomFor）

当你要追加 addlen 字节，如果 alloc - len < addlen，则需要扩容。典型策略：

- 若新长度 **小于阈值 1MB**（SDS_MAX_PREALLOC），采用 **倍增**（通常约=当前需要的两倍）；

- 若新长度 **≥ 1MB**，则按 **所需长度 + 1MB** 预分配。

  这样做保证了**摊还 O(1) 追加**，且避免超大块的指数级增长。

>

|>:

很多截断/切片操作只会降低 len，**不改 alloc**（例如 sdsclear 仅把 len=0 并保留容量），方便之后继续追加时复用空间，减少 realloc。



## **主动收缩**

需要回收多余内存时，可调用：

- sdsRemoveFreeSpace(s)：把 alloc 缩到 len；
- 或 sdsTryResize(s, newlen)：尝试按目标容量调整。





# **5. 常用 API（语义与“坑”）**

> 注：SDS 里很多操作**可能返回新指针**（因为 realloc 可能移动地址），所以要写成 s = sdscat(s,"...");，别丢了返回值。

**创建/释放**

- sds s = sdsnewlen(init, initlen);
- sds s = sdsnew("hello");
- sdsfree(s);



**长度/容量**

- size_t sdslen(s); —— O(1)
- size_t sdsavail(s); —— 剩余可用空间 = alloc - len
- sdsclear(s); —— len=0 但不缩容



**追加/拷贝**

- s = sdscatlen(s, p, n);
- s = sdscat(s, "world");
- s = sdscpylen(s, p, n);（覆盖式拷贝）



**修剪/区间**

- s = sdstrim(s, " \r\n\t"); —— 去两端字符集合
- s = sdsrange(s, start, end); —— 就地切片，超界会裁剪



**小工具**

- s = sdsgrowzero(s, len); —— 把字符串扩到指定长度，中间用 \0 补零（常配合二进制协议）
- sdsupdatelen(s); —— 当你**自己**直接写入 buf 后，用它重算 len（依赖 strlen，二进制内容不该用它）
- sdsIncrLen(s, incr); —— 在已预留空间的前提下，**手动**增加/减少 len（读网络数据后非常高效）

**容易踩坑的点**

1. **忘记接收返回值**：很多函数可能 realloc，**必须**写 s = ...。
2. **没扩容就写**：把 SDS 当成普通 char* 乱 strcat/strcpy 会越界；先 sdsMakeRoomFor 或使用 SDS 追加 API。
3. **二进制数据误用 sdsupdatelen**：它内部用 strlen，遇到中间 \0 会截断。
4. **多指针别名失效**：扩容后老指针失效，别把 s、header、buf 的中间指针长期缓存。
5. **线程安全**：SDS 本身**不**线程安全（Redis 主线程模型确保安全；模块/外部使用需要自己同步）。







# **6. 与 Redis 对象体系（**robj）的关系

Redis 的字符串对象有两种常见**编码**：

- **RAW**：ptr 指向一个普通 SDS（独立分配）；适合可变、较大的字符串。
- **EMBSTR**：**小字符串**的“嵌入式”布局：robj + sds header + buf **一次性分配**成连续内存，降低分配/释放次数、提升局部性。
  - 64 位构建里阈值通常是 **44 字节**（OBJ_ENCODING_EMBSTR_SIZE_LIMIT）。超过会转 RAW；对 EMBSTR 做修改也会转 RAW（因为嵌入布局不可原地增长）。

这层设计让小值高效（减少 malloc 次数与碎片），大值则用常规 SDS 保持灵活性。





# **7. 扩容策略与复杂度直觉小例**

假设从空串开始，每次追加 100B：

- 第 1 次：需要 100B，容量从 0 涨到 ~200B（倍增），摊还成本低；

- …累计到接近 1MB 前基本是倍增；

- 之后每次需要更多时，按“所需 + 1MB”的策略线性前行。

  这就保证了**总体追加是线性时间**，单次摊还接近 O(1)。







# **8. 与 Java** **StringBuilder**的类比（便于上手）

SDS ≈ C 世界的 StringBuilder：

- 都有 **capacity / length** 的区分；
- 都有 **扩容策略** 与 **摊还 O(1) 追加**；
- 不同点：SDS 是 **字节序列**（不做字符集语义），且暴露底层内存，需要你遵守 API 约定（指针可能移动、二进制安全、自己控制写入长度）。







# **9. 一段“正确用法”的 C 示例（体现关键习惯）**

```
sds s = sdsnew("foo");          // len=3
s = sdscat(s, "bar");           // 可能realloc，必须接返回值
// s == "foobar"
size_t n = 1024;
s = sdsMakeRoomFor(s, n);       // 预留至少 n 字节
read(fd, s + sdslen(s), 512);   // 直接把数据写到尾部空闲区（已预留）
sdsIncrLen(s, 512);             // 正确更新 len（O(1)）
/* ...处理... */
sdsfree(s);
```



# **10. 小结（Checklist）**

- 返回类型是 char*，但**背后有头部**；读头用 s[-1] + 指针回跳（库已封装，别自己搞）。
- 记住三元组：len / alloc / buf[len]=='\0'。
- 追加要用 SDS API（或先 sdsMakeRoomFor 再 sdsIncrLen）。
- **任何可能扩容的 API 都要接回返回值**。
- 二进制内容别用基于 strlen 的函数更新长度。
- 与 Redis 对象编码：小字符串 **EMBSTR**，大/可变字符串 **RAW(SDS)**。