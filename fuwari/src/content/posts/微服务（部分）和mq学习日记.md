---
title: 微服务（部分）和mq学习日记
published: 2026-09-17
description: ''
image: ''
tags: [笔记,java]
category: '笔记'
draft: false 
lang: ''
---
- [x] 是否借助ai


# hmall 微服务项目排错与知识点总结

> 一次完整的踩坑链：网关缺类启动失败 → RabbitMQ 认证/vhost → 购物车 user_id 为 null →
> Feign 跨服务丢失用户 → 复制 Maven 模块不识别。
> 贯穿主题：**依赖作用域、条件装配、用户信息在多次"进程边界"上的显式传递**。

---

## 一、问题时间线（每个错误的根因一句话）

| # | 现象 | 根因 | 修复手段 |
|---|------|------|----------|
| 1 | 网关启动 `ClassNotFoundException: MessageConverter` | hm-common 的 `MqConfig` 被无条件自动装配，但 amqp 依赖是 provided，网关没有该类 | `MqConfig` 加 `@ConditionalOnClass` |
| 2 | trade-service 启动 `ACCESS_REFUSED ... PLAIN` | RabbitMQ 不存在 hmall 用户或密码不对 | `rabbitmqctl add_user / change_password` |
| 3 | trade-service 报 `530 vhost /hmall not found`（服务能启动但不断重试） | 虚拟主机未创建、未授权 | `add_vhost` + `set_permissions` |
| 4 | cart-service 加购报 `Field 'user_id' doesn't have a default value` | 网关只打印 userId 没下传；下游无拦截器，ThreadLocal 为 null | 网关 mutate 加头 + 下游 UserInfoInterceptor + MvcConfig |
| 5 | 余额支付 Feign 调 user-service 返回 500 | Feign 是全新 HTTP 请求，不会自动透传请求头，下游 UserContext 又为 null | 自定义 `RequestInterceptor` 复制 user-info 头 |
| 6 | 复制进来的 publisher 文件夹不是模块 | 未登记 `<modules>`；且子 pom 的 parent 还指向旧项目 mq-demo | 改 parent、补依赖、登记模块、Reload Maven |

---

## 二、Maven 依赖机制

### 1. classpath（类路径）

- JVM 运行时查找所有 .class / jar 的"路径清单"，IDEA 启动命令里的 `-classpath a;b;c.jar` 就是它。
- 清单里没有的类，运行时就抛 `ClassNotFoundException`。
- **每个服务的 classpath 相互独立**。
- 判断缺类问题的第一现场：在启动命令的 classpath 里搜报错类所在的 jar。

### 2. 依赖范围 scope

| scope | 编译可见 | 运行时在 classpath | 向下游传递 | 典型用途 |
|---|---|---|---|---|
| compile（默认） | 是 | 是 | 是 | 普通依赖 |
| provided | 是 | **否** | **否** | servlet-api（外部 Tomcat 提供）、hm-common 里的 amqp/mybatis |

- provided 的语义："编译时借我看，运行环境会自备，不要打包、不要传递"。
- 微服务公共模块中"可选功能"依赖（MQ、数据库、webmvc）应设 provided，由**真正使用的服务自己引入 starter**。

### 3. 依赖传递

- A 依赖 B（compile），B 依赖 C（compile），则 A 的 classpath 自动有 C。
- provided 在传递链上会被"掐断"。
- 本项目实际链路：
  - 网关 → hm-common，amqp 是 provided → **拿不到 MessageConverter**
  - trade/pay-service 自己声明 `spring-boot-starter-amqp`（compile）→ starter 传递整套 amqp jar → **正常**

### 4. 复制外部 Maven 模块的标准手续（三步，缺一不可）

1. **改子模块 pom 的 `<parent>`**：指向当前聚合工程（从旧项目复制来的 parent 会导致 Non-resolvable parent POM），并补齐原父 pom 继承来的依赖；
2. **在根 pom 的 `<modules>` 中登记** `<module>publisher</module>`；
3. **IDEA 重新导入**：Maven 面板刷新，或 pom 右键 Add as Maven Project。

> 复制文件 ≠ 加入工程。蓝色方块模块图标 = 已被 Maven 识别。

---

## 三、Spring Boot 自动装配

### 1. spring.factories

- 路径：`src/main/resources/META-INF/spring.factories`
- 作用：公共 jar 里的配置类不在使用方的扫描包（`@SpringBootApplication` 所在包及子包）内，必须在此"登记户口"才会被自动装配。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.hmall.common.config.MyBatisConfig,\
  com.hmall.common.config.MqConfig,\
  com.hmall.common.config.JsonConfig,\
  com.hmall.common.config.MvcConfig
```

- 包内的配置类（如 hm-service 单体里的 config）能被直接扫描，不需要登记。

### 2. @ConditionalOnClass —— 可选依赖的标准兜底

```java
@Configuration
@ConditionalOnClass(MessageConverter.class)
public class MqConfig {
    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
}
```

- 含义：括号中的类在 classpath 存在时，整个配置类才生效，否则跳过。
- 为什么不会因为引用了缺失类而崩：Spring Boot 用 ASM **先读字节码判断条件，再决定是否加载类**，类不加载就不会解析其方法签名。
- 触发背景：JVM 加载类时必须解析其所有方法的参数/返回值类型；反射 `getDeclaredMethods()` 时找不到类型即 `NoClassDefFoundError`。
- hm-common 的统一套路：

| 配置类 | 条件类 | 防住谁 |
|---|---|---|
| MyBatisConfig | MybatisPlusInterceptor、BaseMapper | 无数据库服务/网关 |
| MqConfig | MessageConverter | 无 MQ 服务/网关 |
| MvcConfig | DispatcherServlet、HttpServletRequest | WebFlux 网关（无 Servlet API） |

- 经验法则：**写进 spring.factories 的配置类，只要引用了 provided 依赖的类型，就必须配 `@ConditionalOnClass`。**

---

## 四、RabbitMQ 排错三连

连接认证是分步的，看错误码区分：

1. TCP 不通：连接超时/拒绝（查容器、端口、防火墙）
2. **用户名密码错**：`ACCESS_REFUSED - Login was refused using authentication mechanism PLAIN`
3. **vhost 不存在**：`reply-code=530, NOT_ALLOWED - vhost /hmall not found`
4. vhost 无权限：`access to vhost ... refused`（三道关卡的最后一道）

常用命令：

```bash
# 用户
docker exec -it mq rabbitmqctl add_user hmall 123
docker exec -it mq rabbitmqctl change_password hmall 123
docker exec -it mq rabbitmqctl list_users
# 虚拟主机
docker exec -it mq rabbitmqctl add_vhost /hmall
docker exec -it mq rabbitmqctl list_vhosts
# 授权：configure / write / read 三个正则
docker exec -it mq rabbitmqctl set_permissions -p /hmall hmall ".*" ".*" ".*"
docker exec -it mq rabbitmqctl list_permissions -p /hmall
```

- `Socket closed` 多为服务端拒绝后主动关闭，不一定是网络问题。
- guest 默认只能 localhost 登录，跨机访问必须自建用户。
- vhost 类似 MySQL 的 database：逻辑隔离队列/交换机/权限。
- 监听容器行为差异：认证失败=致命错误，启动直接失败；vhost/网络故障=可恢复错误，应用启动成功但后台每 5 秒重试（功能仍不可用）。

---

## 五、ThreadLocal 与用户信息透传

### 1. UserContext

```java
public class UserContext {
    private static final ThreadLocal<Long> tl = new ThreadLocal<>();

    public static void setUser(Long userId) { tl.set(userId); }
    public static Long getUser() { return tl.get(); }
    public static void removeUser() { tl.remove(); }
}
```

- 作用域：**同一 JVM + 同一线程**，本质是线程私有 Map。
- `afterCompletion` 必须 `remove()`：Tomcat 线程池复用线程，不清理会"用户串号"。

### 2. 核心原则

> ThreadLocal 跨不了进程。凡是跨进程边界（HTTP、MQ），用户信息都必须**显式搭车传输**，每一跳都要有人搬运；HTTP 协议本身无状态，框架不会自动把头传给"下一个请求"。

### 3. 用户信息的完整传递链

```
浏览器 --JWT--> 网关验签得到 userId
网关 --(加 user-info 请求头)--> 微服务
微服务拦截器读头 -> UserContext.setUser()
微服务A --Feign 新 HTTP 请求(需 RequestInterceptor 复制头)--> 微服务B
微服务B 拦截器读头 -> 重建 UserContext
```

### 4. 第一环：网关注入请求头（WebFlux，对象不可变）

```java
// 把 userId 放入名为 user-info 的请求头
ServerHttpRequest newRequest = request.mutate()
        .header("user-info", userId.toString())
        .build();
ServerWebExchange newExchange = exchange.mutate().request(newRequest).build();
return chain.filter(newExchange);
```

- WebFlux 的 request/exchange 是**不可变对象**（响应式并发安全），没有 set 方法。
- `mutate()` = 建造者模式：以原对象为蓝本复制 → 修改 → `build()` 产出新对象，原对象不变（类比 String）。
- exchange 是请求在网关中的上下文容器，`chain.filter()` 只收 exchange，所以要把新 request 装回新 exchange。

### 5. 第二环：下游微服务接收（Spring MVC 拦截器）

```java
public class UserInfoInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 1.从请求头取网关传来的用户信息
        String userInfo = request.getHeader("user-info");
        // 2.非空则存入 ThreadLocal
        if (StrUtil.isNotBlank(userInfo)) {
            UserContext.setUser(Long.valueOf(userInfo));
        }
        // 3.放行
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse resp, Object handler, Exception ex) {
        // 4.请求结束清理，避免线程池复用导致用户串号
        UserContext.removeUser();
    }
}
```

### 6. MvcConfig：注册拦截器

```java
@Configuration
@ConditionalOnClass({DispatcherServlet.class, HttpServletRequest.class})
public class MvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new UserInfoInterceptor());
    }
}
```

- `WebMvcConfigurer`：Spring MVC 留给开发者的扩展插槽，重写回调即可定制默认配置。
- 拦截器在 Controller 之前执行，先把 userId 放入 ThreadLocal，Service 层才取得到。
- 放 hm-common + 登记 spring.factories → 所有微服务复用；`@ConditionalOnClass` 防网关（WebFlux 无 DispatcherServlet）缺类崩溃。
- 执行顺序：`preHandle → Controller/Service → afterCompletion`。

### 7. 第三环：Feign 跨服务透传

```java
@Bean
public RequestInterceptor userInfoRequestInterceptor() {
    return template -> {
        // 1.拿到当前线程正在处理的"入站请求"
        ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attrs == null) {
            // 没有请求上下文（MQ 监听器、定时任务等异步线程），直接跳过
            return;
        }
        // 2.从入站请求头里取出网关注入的 user-info
        String userId = attrs.getRequest().getHeader("user-info");
        // 3.写入 Feign 即将发出的"出站请求"头里
        if (StrUtil.isNotBlank(userId)) {
            template.header("user-info", userId);
        }
    };
}
```

- Feign 是通用 HTTP 客户端，每次调用构造**全新请求**，默认不复制入站请求头。
- `RequestContextHolder`：Spring MVC 基于 ThreadLocal 持有"当前入站请求"。
- `attrs == null` 判断：异步线程（MQ、`@Async`、线程池）没有入站请求。
- 配置位置：hm-api 的 `DefaultFeignConfig`（经 `@EnableFeignClients(defaultConfiguration=...)` 全局生效）。
- hm-api 若编译找不到 `HttpServletRequest`，加 provided 的 `tomcat-embed-core`。
- 推论：**MQ 消息不透传用户身份**（收发时间/线程完全不同），需要用户时从业务数据（如订单的 bizUserId）取。

---

## 六、异常栈（Stack Trace）排查方法论

固定四步：

1. **找根因**：搜 `with root cause` / 最底层 `Caused by`，读那句"人话"。
2. **看现场**：MyBatis 日志的 `Preparing`（SQL）+ `Parameters`（实际参数），定位什么数据不对。
   - 例：INSERT 列清单缺 user_id + 查询参数是 null → 上游传值缺失，而非数据库问题。
3. **定位代码**：栈中只找**自己的包名**（com.hmall.*），框架包（org.springframework.* 等）跳过；
   IDEA 中 Ctrl+点击蓝色链接，或 Ctrl+G 输行号。
   - 注意区分："异常炸点行"（如 save()）≠"错误数据出生地"（如 getUser() 返回 null），要带着问题向上追。
4. **反向追问边界**：这个值谁应该赋值？沿调用链一直追到断链处（网关头丢失 / Feign 未透传）。

微服务特有原则：

> 调用方日志里的 FeignException/500 只说明"对方挂了"，**真正的异常栈要去被调服务的控制台找**。

其他技巧：

- 异常栈从上往下是"抛出点→调用者"，从下往上是请求实际流动方向。
- 看到光秃秃的 Spring 默认错误 JSON（timestamp/status/error/path）而非统一返回，通常是全局异常处理器 `@RestControllerAdvice` 未被扫描/未注册 spring.factories。

---

## 七、其他零散知识点

- **spring.factories 自动配置 vs 组件扫描**：前者管 jar 包外配置类，后者管启动包子包。
- **@Configuration**：声明配置类；但包外的配置类单靠它不会被发现，仍需 spring.factories。
- **WebFlux(网关) 与 WebMVC(业务服务) 不能混**：网关 classpath 有 spring-webflux，加载 servlet 相关类会失败——这也是 hm-common 所有配置类都要条件化的原因。
- **MyBatis-Plus 默认 NOT_NULL 插入策略**：值为 null 的字段不进 INSERT 列清单，数据库 NOT NULL 无默认值时即报错。
- **Feign 调试**：`Logger.Level.FULL` 可打印完整请求/响应（头、行、体），跨服务排错利器。
- **改公共模块后必须 Rebuild / mvn install**：下游运行时用的是 jar/target 编译产物。

---

## 八、本次涉及的关键文件清单

| 文件 | 作用 |
|---|---|
| `hm-common/.../config/MqConfig.java` | RabbitMQ 消息转换器，加 @ConditionalOnClass |
| `hm-common/.../config/MvcConfig.java` | 注册 UserInfoInterceptor（新建） |
| `hm-common/.../interceptor/UserInfoInterceptor.java` | 下游读取 user-info 头入 ThreadLocal（新建） |
| `hm-common/resources/META-INF/spring.factories` | 公共自动装配登记处 |
| `hm-gateway/.../filters/AuthGlobalFilter.java` | JWT 验签 + mutate 透传 user-info |
| `hm-api/.../config/DefaultFeignConfig.java` | Feign 日志级别 + 请求头透传拦截器 |
| `cart-service/.../CartServiceImpl.java` | 修复硬编码 userId=1 的 TODO |
| `publisher/pom.xml`、根 `pom.xml` | 外部模块迁入：改 parent、补依赖、登记 modules |

---

## 九、一句话总纲

**公共模块靠"provided 依赖 + @ConditionalOnClass + spring.factories"做到按需装配；
用户身份靠 ThreadLocal 在进程内保存、靠 HTTP 请求头在每个进程边界被显式搬运（网关 mutate、MVC 拦截器、Feign RequestInterceptor 各管一跳）；
排错时先找 root cause 与 SQL 参数，再沿自己的包名追代码，跨服务就去对方日志里找真相。**
