### #1.1.1

#### 隐式拆箱导致 NEP

```java
Integer x = null;
int y = x;        // 这里 NPE：x.intValue()

Boolean ok = null;
if (ok) {}        // NPE：ok.booleanValue()

Integer code = null;
switch (code) {   // 隐式拆箱 → NPE
  ...
}
```

#### 包装数据类型缓存

```java
Integer a = 127, b = 127;    a == b  // true（缓存）
Integer c = 128, d = 128;    c == d  // false（不同对象）
Objects.equals(c, d)         // true（值相等）
```

#### 重载与装箱的选择规则

```java
//一般优先级：精确匹配 > 基本类型宽化（int→long） > 装箱 > 可变参数
void f(long x) {}
void f(Integer x) {}
f(1);   // 选 f(long)（宽化优先于装箱）

void g(int x) {}
void g(Integer x) {}
g(null);   // 选 g(Integer)（基本类型不能接 null）

void h(Integer x) {}
void h(Long x) {}
h(null);   // 编译错误：不明确
```

#### String 不可变

- 设计目的：**线程安全共享**、**可做 Map/Set key**、**常量池复用**、**安全性（类加载/URL 等）**、**可缓存 hash**。

- 实现要点（JDK 9+）：String 内部是 byte[] + coder(Latin1/UTF-16)，内容不变；hash 懒计算后缓存。

  【关键词】String immutability、hash 缓存、Compact Strings(JDK9+)

#### 字符串常量池

- **字面量**（"abc"）在编译期入常量池，JVM 启动后加载到堆内的全局字符串池，**同值同实例**。

- new String("abc") 一定创建新对象；intern() 返回池中“规范实例”（无则放入并返回它）。

- 池可减少重复对象，但**不要滥用** **intern()** **去收大批动态内容**（增 GC 压力）。

  【关键词】string pool、intern()、字面量 vs new

```java
String a = "ab", b = "ab";      a == b        // true（同一池中实例）
String c = new String("ab");    a == c        // false ，单看这一步会创建 1 or 2个对象（常量池一个、String 类一个）
String d = c.intern();          a == d        // true
```

#### 字符串拼接优化

编译器优化

```java
static final String A = "he", B = "llo";
String s = A + B;          // 编译后就是 "hello"
```

运行期优化

- **JDK 8 及之前（普遍认知）**：a + b + c 编译成 new StringBuilder().append(a).append(b)...toString()

- **JDK 9+（更现代）**：编译成 invokedynamic 调用 StringConcatFactory，JIT 可能生成更优代码（不一定真用 StringBuilder）。

- 结论：**单个表达式里的** **+** **在 JDK 9+ 写着就好**；**循环累加**仍要小心（见下）。

  【关键词】invokedynamic、StringConcatFactory、+ vs StringBuilder

#### Objects.equals(a, b)

- 等价逻辑：

```java
(a == b) || (a != null && a.equals(b))
```

- 好处：**不 NPE**，且对 a==b 的快速路径做了优化。
- 适用：任意可能为 null 的引用比较；Map/Set 自定义 key 比较时也建议用它。

```java
new BigDecimal("1.0").equals(new BigDecimal("1"));     // false（规模不同）
new BigDecimal("1.0").compareTo(new BigDecimal("1"));  // 0（数值相等）
```

### #1.4.1

#### 擦除式泛型（erasure-based generics）

- **Java 是“擦除式泛型”（erasure-based generics），不是“假泛型”。**

  它在**编译期**用泛型做完整的类型检查、推断和约束；在**运行期**把绝大多数泛型实参擦除成上界（无上界→Object，<T extends Number>→Number），因此对象层面没有“List<String> 与 List<Integer> 的区别”。

**编译期 vs 运行期**

- **编译期会发生什么**

  1. **类型检查/推断**：List<String> 只能放 String；泛型方法 <T> T first(List<T>) 会推出 T。
  2. **插桩**：必要时插入强转（避免你手写 (String)）、生成**桥接方法（bridge method）**保证覆写一致性。
  3. **通配符/边界**：? extends / ? super 的规则在此阶段生效（为什么 List<? extends Number> 不能 add(1) 就是这里决定的）。

  

- **运行期会发生什么**

  1. **类型被擦除**：new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass() 为 true。
  2. **不能做的事**：instanceof List<String>、new T()、new T[]、在运行期区分 List<String> vs List<Integer>。
  3. **数组与泛型差异**：数组是**协变 + 运行期带元素类型**（可能抛 ArrayStoreException），泛型是不变 + 运行期擦除（更安全，错误在编译期暴露）。

> 小结：编译期**真类型安全**，运行期**看不到实参类型**（对象层面）。

**两个“保留信息”的例外（不是再化，但能“看见点东西”）**

1. **类文件里保留了“泛型签名”元数据**（Signature 属性），反射能在“**声明处**”读到它：
   - 能读到：字段/方法/类的**声明**上的 List<String>、Map<Long,User>。
   - 读不到：**局部变量**的泛型实参、对象“当前装的是什么类型”。

```java
class C {
  List<String> names; // 反射：field.getGenericType() 可见 <String>
}
var list = new ArrayList<String>();
list.getClass();      // 只得到 ArrayList.class，看不到 <String>
```

```java
List<? extends Number> src = List.of(1, 2L);
Number n = src.get(0); // ✅
src.add(1);            // ❌ 编译错（除了 null 之外都不安全）

List<? super Integer> dst = new ArrayList<Number>();
dst.add(1);       // ✅
Object x = dst.get(0); // 只能当 Object 取出

List<String> ls = new ArrayList<>();
List raw = ls;       // ⚠️ 未检查的转换（unchecked），编译器会给 warning
raw.add(123);        // ✅ 编译通过（因为 raw 上的 add 参数类型是 Object）

String s = ls.get(0); // 💥 运行期 ClassCastException: Integer cannot be cast to String
```

泛型实际使用

```java
public static <T> List<Long> copyIds(List<? extends T> src, Function<? super T, ? extends Long> idFn) {
    List<Long> out = new ArrayList<>(src.size());
    for (T t : src) {
        Long id = idFn.apply(t);
        if (id != null) out.add(id); // 这里选择跳过 null
    }
    return out;
}

List<Long> ids1 = copyIds(users, User::getId);
System.out.println(ids1); // [1, 3]
```

