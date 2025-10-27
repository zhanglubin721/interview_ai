# **心智模型（两条线）**

- **容器级启动线**：注册 BeanDefinition → 运行 *BeanFactoryPostProcessor* → 注册/排序 *BeanPostProcessor* → 实例化单例 Bean → 刷新完成后的回调。
- **单个 Bean 的生命周期线**：实例化 → 属性注入 → Aware 注入 → 初始化回调 → 代理包装 → 放入一级缓存 →（容器刷新完成后的）单例回调 → 运行期 → 销毁。





# **单个 Bean 的完整时间线（singleton；按源码顺序）**

> 主要路径：doCreateBean → populateBean → initializeBean；关键钩子在注释里

1. **类型预测/构造器选择（还没创建对象）**

   - SmartInstantiationAwareBeanPostProcessor：

     - predictBeanType（预测最终类型）
     - determineCandidateConstructors（挑构造器；支持 @Autowired/必选参数等）
     - getEarlyBeanReference（仅在循环依赖需要 early-ref 时会用到，提前给出“可能已被代理”的引用）

     

   

2. **实例化（createBeanInstance）**

   - 执行构造器/工厂方法；此时**只得到“裸对象”**（target object）
   - 如果允许循环依赖：把一个 ObjectFactory 放进**三级缓存**（singletonFactories），**按需**能产出“早期引用”（可能是代理）

   

3. **属性注入（populateBean）**

   - InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation（可阻止后续注入）
   - 解析依赖（@Autowired / @Qualifier / @Value / @Resource）并注入字段/Setter
   - InstantiationAwareBeanPostProcessor#postProcessProperties（替代旧的 postProcessPropertyValues）

   

4. **Aware 回调（invokeAwareMethods）**

   - 若实现：BeanNameAware / BeanClassLoaderAware / BeanFactoryAware 等，注入容器元数据

   

5. **初始化前置（applyBeanPostProcessorsBeforeInitialization）**

   - BeanPostProcessor#postProcessBeforeInitialization
   - **@PostConstruct** 就在这里由 CommonAnnotationBeanPostProcessor 调用（**先 @PostConstruct**）

   

6. **初始化（invokeInitMethods）**

   - InitializingBean#afterPropertiesSet()（**在 @PostConstruct 之后**）
   - 自定义 init-method（在 afterPropertiesSet() 之后）

   

7. **初始化后置（applyBeanPostProcessorsAfterInitialization）**

   - BeanPostProcessor#postProcessAfterInitialization
   - **自动代理创建器（如** **AnnotationAwareAspectJAutoProxyCreator****）在这里“统一”决定是否代理，创建**“一个最终代理”**并返回**
   - 完成后把（可能是代理的）**成品**放入**一级缓存**（singletonObjects）

   

8. **容器刷新完成后的单例回调**

   - SmartInitializingSingleton#afterSingletonsInstantiated()（当所有非懒加载单例都就绪后一次性回调；常用于“需要其他单例都 ready”再做的事）

   

9. **运行期**

   - 如果被 AOP：每次方法调用经过 **拦截器链**（事务/校验/日志等），不是套娃代理，是**一个代理 + 一条链**

   

10. **销毁（关闭容器时，单例才触发）**

- DestructionAwareBeanPostProcessor#postProcessBeforeDestruction()（**这里触发** **@PreDestroy**）
- DisposableBean#destroy()
- 自定义 destroy-method

> 顺序就是：**@PreDestroy → DisposableBean#destroy → 自定义 destroy-method**

> ⚠️ prototype Bean：容器**不会**自动走销毁回调；需要你自己管理。







# **容器级启动关键点（影响“什么时候创建 Bean”和“谁先注册处理器”）**

1. **注册 BeanDefinition**：扫描/导入配置类、组件扫描、@Import 等

2. **BeanDefinitionRegistryPostProcessor**（比 BFPP 更早）：

   典型如 ConfigurationClassPostProcessor，会把 @Configuration / @Bean 等解析成新的 BeanDefinition

3. **BeanFactoryPostProcessor****（BFPP）**：

   典型 PropertySourcesPlaceholderConfigurer（处理 ${...} 占位符）；这一步还没创建目标 Bean

4. **注册并排序所有** **BeanPostProcessor**：

   包括 CommonAnnotationBeanPostProcessor、AOP 的 AutoProxyCreator、校验、异步等——**它们决定后续初始化阶段的行为**

5. **实例化非懒加载单例**：按依赖拓扑创建；此时走上面那条“单个 Bean 生命周期线”

6. **发布** **ContextRefreshedEvent**、调用 SmartInitializingSingleton





# **AOP/代理与生命周期的相对位置（很容易混淆）**

- **@PostConstruct**：在 postProcessBeforeInitialization 阶段由 CommonAnnotationBeanPostProcessor 执行，**早于** afterPropertiesSet。
- **代理创建**：在 postProcessAfterInitialization 阶段，**一次把所有 Advisor 安装到一个代理里**；若有循环依赖可能会在 **early-ref** 时提前创建“同一个最终代理”。
- **自调用问题**：无论何时创建了代理，**同类内部** **this.method()** **调用不会走代理**，因此不会进事务/校验切面。





# **FactoryBean 与普通 Bean 的差异**

- 取 getBean("foo")：如果 foo 是 FactoryBean，返回它**生产的对象**（FactoryBean#getObject()）；
- 取 getBean("&foo")：才是取到 FactoryBean 自身。
- 生命周期：先创建 FactoryBean 自身，它也会走上面的生命周期；随后按需创建产品对象。





# **常见顺序表（背下来就不乱了）**

**初始化阶段（从早到晚）**

1. @Autowired / 依赖注入完成
2. Aware：BeanNameAware → BeanClassLoaderAware → BeanFactoryAware → …
3. BeanPostProcessor#postProcessBeforeInitialization（含 **@PostConstruct**）
4. InitializingBean#afterPropertiesSet
5. 自定义 init-method
6. BeanPostProcessor#postProcessAfterInitialization（**AOP 代理**在这里产生；一次成型）



**销毁阶段（从早到晚）**

1. DestructionAwareBeanPostProcessor#postProcessBeforeDestruction（含 **@PreDestroy**）
2. DisposableBean#destroy
3. 自定义 destroy-method





# **作用域与时机**

- **singleton**：默认；refresh() 期间（非懒加载）被创建；关闭容器时统一销毁。
- **prototype**：每次 getBean 都新建；**无销毁回调**。
- **request/session/application**（Web）：随作用域生命周期；销毁由作用域管理器触发。





# **几个“易错点 / 面试高频”**

- **构造器 ↔ 构造器循环依赖**：三级缓存救不了；字段/Setter可解（见你前面的问题）。
- **@PostConstruct** **与** **afterPropertiesSet** **顺序**：**先** **@PostConstruct**，再 afterPropertiesSet，再 init-method。
- **代理只有一个**：多个 Advisor 通过 ProxyFactory **一次组装**；运行期走**拦截器链**。
- **prototype** **不会自动销毁**：很多人以为 @PreDestroy 会触发——不会。
- **SmartInitializingSingleton** **的触发时机**：所有非懒加载单例创建并初始化完毕之后，适合“跨 Bean 的最终协调”。
- **FactoryBean** **的取法**："name" 取产品；"&name" 取工厂。





# **最小可观察示例（打印顺序）**

```
@Component
public class DemoBean implements BeanNameAware, InitializingBean, DisposableBean {
  @PostConstruct void post() { log.info("postConstruct"); }
  @Override public void setBeanName(String n) { log.info("beanNameAware"); }
  @Override public void afterPropertiesSet() { log.info("afterPropertiesSet"); }
  public void init() { log.info("customInit"); }   // 配置 init-method="init"
  @PreDestroy void preDestroy() { log.info("preDestroy"); }
  @Override public void destroy() { log.info("destroy"); }  // 配置 destroy-method 也可
}
```

**期望日志顺序（初始化）**：beanNameAware → postProcessBeforeInitialization(…包含@PostConstruct) → afterPropertiesSet → customInit → postProcessAfterInitialization(…可能AOP)

**销毁**：postProcessBeforeDestruction(…包含@PreDestroy) → destroy → customDestroy



# 面试问答：简述 Spring Bean 的生命周期

**Spring Bean（以单例为例）在容器刷新时先被“实例化”（选构造器/工厂方法得到裸对象），随后做“依赖注入/属性赋值”，接着触发各类 Aware 回调（如 BeanNameAware/BeanFactoryAware），然后进入“初始化前置”——BeanPostProcessor#postProcessBeforeInitialization（此阶段会执行 @PostConstruct），再到“初始化”本身——InitializingBean#afterPropertiesSet 与自定义 init-method，紧接着“初始化后置”——BeanPostProcessor#postProcessAfterInitialization（此步若命中切面会**一次性创建最终代理**）；完成后放入一级缓存供业务使用；当容器关闭时按顺序销毁：DestructionAwareBeanPostProcessor#postProcessBeforeDestruction（执行 @PreDestroy）→ DisposableBean#destroy → 自定义 destroy-method。补充：prototype 不自动执行销毁回调；所有单例就绪后还会触发 SmartInitializingSingleton#afterSingletonsInstantiated 用于“全局就绪后”的初始化收尾。