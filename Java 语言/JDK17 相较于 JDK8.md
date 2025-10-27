# **1) 语言特性：写代码更省心（8→17 的质变）**

- **var** **局部变量类型推断（J10）**

```
var map = new HashMap<String, Integer>(); // 编译期推断，非动态类型
```

- **switch** **表达式（J14）**：更简洁、可返回值

```
int days = switch (month) {
  case JAN, MAR, MAY -> 31;
  case APR, JUN     -> 30;
  default           -> { yield 28; }
};
```

- **record（J16）**：轻量不可变 DTO，自动生成构造/equals/hashCode/toString

```
public record User(long id, String name) {}
```

- **sealed class（J17）**：受控继承层次 + 编译期穷尽性检查

```
public sealed interface Shape permits Circle, Rect {}
```

- **instanceof** **模式匹配（J16）**：免去显式强转

```
if (obj instanceof String s) { System.out.println(s.length()); }
```

- **文本块 Text Blocks（J15）**：多行字符串，写 SQL/JSON 友好

```java
String sql = """
    SELECT id, name
    FROM user
    WHERE name LIKE ?
    """;
```

- **更“有用”的 NPE（J14）**：异常信息直接标出哪个变量是 null（默认开启）。

> 面试话术：**“JDK 17 让日常 Java 写法更现代：record 做 DTO、sealed 控层次、switch 表达式和 instanceof 匹配把样板代码砍掉，文本块写 SQL/JSON 清爽不少。”**





# **2) 平台/性能：GC、容器、启动与诊断全面升级**

- **GC 选择更多，暂停更稳**

  - JDK 8 默认 Parallel；**JDK 9 起默认 G1**，17 上 G1 已很成熟（并发类卸载、混合回收优化等）。
  - 17 可选 **ZGC** / **Shenandoah**（低暂停 GC，目标 <10ms 级别），对低延迟服务很友好。
  - 还有 **Epsilon**（No-Op）用于基准测试。

- **容器感知**（J10+）：自动识别 cgroup 限额；用 -XX:MaxRAMPercentage=75.0 等更好地控制容器内内存。

- **CDS/AppCDS**：类元数据共享，**更快启动、更省内存**（微服务场景收益明显）。

- **JFR（Java Flight Recorder，J11 开源）**：线上低开销火焰图/事件采集：

  -XX:StartFlightRecording=filename=app.jfr,dumponexit=true

- **TLS 1.3、ChaCha20/Poly1305（J11）**：网络更安全更快。

- **强封装**（J16 起更严格）：JDK 内部包不鼓励反射访问，旧框架可能需要 --add-opens 过渡。

> 面试话术：**“我们在 17 上跑微服务：G1 已经够稳；超低延迟可以切 ZGC。容器里 17 会按 cgroup 算内存，结合 CDS/JFR，启动更快、诊断更强。”**







# **3) 常用新 API：替掉“三方小轮子”**

- **java.net.http.HttpClient****（J11）**：官方异步 HTTP/2

```
var client = HttpClient.newHttpClient();
var req = HttpRequest.newBuilder(URI.create(url)).GET().build();
var body = client.send(req, HttpResponse.BodyHandlers.ofString()).body();
```

- **字符串/文件便捷方法（J11+）**

```
"  a  ".strip();           // 去空白（支持 Unicode）
"a\nb\n".lines().count();  // 行流
"ha".repeat(3);            // "hahaha"
Files.mismatch(p1, p2);    // 返回首个不匹配位置，-1 表示相同
```

- **流程/反压（J9 Flow API）**：与 Reactor/RS 生态对接更规范。
- **CompletableFuture** **增强**、ProcessHandle、更丰富的 Optional 方法等。

> 面试话术：**“JDK 17 自带的 HttpClient 替了老 Apache 客户端；String/Files 新方法满足很多小需求；再加上 CompletableFuture/Flow，写并发和响应式更顺手。”**





## **迁移/兼容要点（Boot/SCA 环境很常见）**

- **Boot 3 / Framework 6 需要 JDK 17 + Jakarta 命名空间**（javax.* → jakarta.*），依赖要跟着换；老反射访问可能要 --add-opens。
- **AOT/JIT 变化**：JDK 17 移除了实验性的 jaotc；如需 Graal AOT/JIT，用 GraalVM 发行版。
- **GC 参数差异**：8 上常见的 CMS 已淘汰，17 上用 G1/ZGC/Shenandoah 替代。







## **一句话总结（面试版）**

**“从 8 升到 17，语言层我能用 record/sealed/switch 表达式/文本块把 DTO、分支和长字符串写得更简洁；平台层我获得 G1/ZGC 等更稳的 GC、容器感知、CDS 和 JFR 带来的启动/诊断优势；API 层我有官方 HttpClient、丰富的 String/Files 工具。升级不是‘只为兼容 Boot/SCA’，而是实际能让代码更短、服务更稳、排障更快。”**