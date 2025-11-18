既然要作为组件提供出去，那么肯定要封装为一个 Spring Boot Starter。

那如何自定义一个 Spring Boot Starter 呢？

老生常谈的问题。

自定义 Starter 就是把业务功能封装成自动配置类，然后在 spring.factories 或者 .imports 中声明，让 Spring Boot 在启动时自动发现并注册。
核心点是：业务类、属性类、自动配置类、条件注解、注册配置文件。

加载流程就是 @EnableAutoConfiguration 触发 SpringFactoriesLoader 读取配置，再由 @Conditional 系列注解判断是否装配 Bean。

内部使用方只需要在 maven 配置中加入一致性行组件的依赖，根据需要修改配置文件即可。

------

### **1. 创建模块**

- 好像 spring 官方推荐 **两模块模式**，不过我看许多开源项目都是一个模块
    
    1. **`xxx-spring-boot-starter`**（Starter 壳模块）  
        只负责依赖管理，依赖 `autoconfigure` 模块
        
    2. **`xxx-spring-boot-autoconfigure`**（自动配置模块）  
        包含配置类、业务类、`spring.factories`/`.imports`
        
- 也可以直接做成单模块，但不利于复用
    

### **2. 编写核心业务类**

- 提供 Starter 对外功能的核心逻辑
    

```java
public class DemoService {
    private final String prefix;
    public DemoService(String prefix) { this.prefix = prefix; }
    public String sayHello(String name) { return prefix + " " + name; }
}
```

---

### **3. 属性绑定类**

- 用于将 `application.yml` 配置绑定到 Java 对象
    

```java
@ConfigurationProperties(prefix = "demo")
public class DemoProperties {
    private String prefix = "Hello";
    // getter/setter
}
```

---

### **4. 自动配置类**

- 使用条件注解，按需装配 Bean
    

```java
@Configuration
@EnableConfigurationProperties(DemoProperties.class)
@ConditionalOnProperty(prefix = "demo", name = "enabled", havingValue = "true", matchIfMissing = true)
public class DemoAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public DemoService demoService(DemoProperties properties) {
        return new DemoService(properties.getPrefix());
    }
}
```

常用条件注解：

- `@ConditionalOnClass` → 类路径存在某个类时生效
    
- `@ConditionalOnMissingBean` → 容器没有这个 Bean 时生效
    
- `@ConditionalOnProperty` → 配置开关控制
    

---

### **5. 注册自动配置类**

- **Spring Boot 2.x**：`META-INF/spring.factories`
    

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.DemoAutoConfiguration
```

- **Spring Boot 3.x**：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
    

```txt
com.example.demo.DemoAutoConfiguration
```

---

## **使用时的执行流程**

1. 项目引入 Starter 依赖
    
2. `@SpringBootApplication` → `@EnableAutoConfiguration` → `AutoConfigurationImportSelector`
    
3. 读取 `spring.factories`/`.imports` 找到自动配置类
    
4. 解析 `@Conditional` 系列注解决定是否生效
    
5. 注册 Bean 到 Spring 容器
    
6. 绑定 `application.yml` 配置到属性类
    
7. Bean 初始化完成，应用可以直接使用
    

