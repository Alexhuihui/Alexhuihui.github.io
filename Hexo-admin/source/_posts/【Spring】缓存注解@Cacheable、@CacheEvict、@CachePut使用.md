---
title: 【Spring】缓存注解@Cacheable、@CacheEvict、@CachePut使用
date: 2024-05-25 19:54:06
categories: Spring
tags: Spring

---

Spring Cache 通过封装统一的缓存抽象层，支持多种缓存实现（如Ehcache、Redis、Caffeine等）。可以使用注解@Cacheable、@CacheEvict、@CachePut等优雅的进行使用，无需修改业务代码。

<!-- more --> 

### @Cacheable

`@Cacheable`注解用于在方法执行前检查缓存，如果缓存中有数据则返回缓存中的数据，否则执行方法并将结果缓存。

**工作原理**：

1. **方法调用前**：
   - AOP代理拦截方法调用。
   - 从指定的缓存区（`value`属性指定）中根据缓存键（`key`属性指定）查找缓存条目。
   - 如果找到缓存条目，则返回缓存的值，不执行目标方法。
2. **方法调用后**：
   - 如果缓存中没有找到条目，则执行目标方法。
   - 将方法返回值存储到缓存中，使用指定的缓存区和缓存键。

**示例**：

```
@Cacheable(value = "users", key = "#userId")
public User getUserById(Long userId) {
    return userRepository.findById(userId).orElse(null);
}
```

### @CacheEvict

`@CacheEvict`注解用于从缓存中移除一个或多个条目，通常在数据修改或删除操作时使用。

**工作原理**：

1. 方法调用后

   ：

   - AOP代理拦截方法调用。
   - 执行目标方法。
   - 从指定的缓存区（`value`属性指定）中根据缓存键（`key`属性指定）移除缓存条目。
   - 如果设置了`allEntries = true`，则清空整个缓存区。

**示例**：

```
@CacheEvict(value = "users", key = "#userId")
public void deleteUserById(Long userId) {
    userRepository.deleteById(userId);
}
```

### @CachePut

`@CachePut`注解用于在方法执行后将结果更新到缓存中。与`@Cacheable`不同，它总是会执行目标方法。

**工作原理**：

1. 方法调用后

   ：

   - AOP代理拦截方法调用。
   - 执行目标方法。
   - 将方法返回值存储到缓存中，使用指定的缓存区（`value`属性指定）和缓存键（`key`属性指定）。

**示例**：

```
@CachePut(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

`@CacheConfig`注解用于在类级别配置缓存的公共设置，简化方法级别的缓存配置。如果类中的多个方法共享相同的缓存配置（例如相同的缓存名称），使用`@CacheConfig`可以避免在每个方法上重复配置。

### @CacheConfig的作用

- **缓存名称**：设置默认的缓存名称，应用于该类中的所有缓存操作。
- **缓存管理器**：指定默认的缓存管理器，应用于该类中的所有缓存操作。

### 示例

假设我们有一个`UserService`类，其中多个方法需要使用相同的缓存配置。

#### 不使用@CacheConfig

```
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Cacheable(value = "users", key = "#userId")
    public User getUserById(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }

    @CacheEvict(value = "users", key = "#userId")
    public void deleteUserById(Long userId) {
        userRepository.deleteById(userId);
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

在这个例子中，每个缓存注解都需要指定`value = "users"`，如果有多个方法共享相同的缓存名称，这样会显得冗余。

#### 使用@CacheConfig

```
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.stereotype.Service;

@Service
@CacheConfig(cacheNames = "users")
public class UserService {

    @Cacheable(key = "#userId")
    public User getUserById(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }

    @CacheEvict(key = "#userId")
    public void deleteUserById(Long userId) {
        userRepository.deleteById(userId);
    }

    @CachePut(key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

在这个例子中：

- `@CacheConfig(cacheNames = "users")`：在类级别配置默认的缓存名称为“users”。
- 各个方法上不再需要重复指定缓存名称，只需要配置`key`属性。

### @CacheConfig 的使用场景

- **统一缓存配置**：当一个类中多个方法使用相同的缓存名称或缓存管理器时，可以通过`@CacheConfig`统一配置，减少代码冗余。
- **简化配置**：避免在每个方法的缓存注解中重复指定相同的缓存名称或其他配置。

### 进一步示例

假设我们在同一个类中使用多个缓存名称，可以通过`@CacheConfig`配置默认的缓存名称，同时在特定方法上覆盖默认配置。

```
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.stereotype.Service;

@Service
@CacheConfig(cacheNames = "users")
public class UserService {

    @Cacheable(key = "#userId")
    public User getUserById(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }

    @CacheEvict(key = "#userId")
    public void deleteUserById(Long userId) {
        userRepository.deleteById(userId);
    }

    @CachePut(key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }

    @Cacheable(value = "admins", key = "#adminId")
    public Admin getAdminById(Long adminId) {
        return adminRepository.findById(adminId).orElse(null);
    }
}
```

在这个例子中：

- `@CacheConfig(cacheNames = "users")`：在类级别设置默认缓存名称为“users”。
- `getAdminById`方法通过`@Cacheable(value = "admins", key = "#adminId")`覆盖了默认的缓存名称，使用“admins”缓存。

通过使用`@CacheConfig`，可以使代码更清晰，避免重复配置，提高可维护性。

### 实现细节

1. **Spring AOP代理**：
   - Spring AOP通过JDK动态代理或CGLIB创建代理对象。
   - 代理对象拦截方法调用，在调用目标方法之前或之后执行缓存逻辑。
2. **CacheManager和Cache接口**：
   - `CacheManager`：Spring缓存抽象中的核心接口，用于管理不同的缓存实现。
   - `Cache`：表示具体的缓存，提供基本的缓存操作方法，如`get`、`put`、`evict`等。
3. **缓存配置**：
   - 在Spring Boot项目中，通过`@EnableCaching`注解启用缓存功能。
   - 在配置文件中指定具体的缓存实现（如Ehcache、Redis等）的配置。

### 示例项目配置

**引入依赖**：

在`pom.xml`中添加依赖：

```
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
```

**配置文件Yaml**：

```
spring:
  application:
    name: buzz-chat

  data:
    redis:
      host: 127.0.0.1
      port: 6379
      database: 0

  cache:
    type: REDIS
```

**主配置类**：

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class CacheApplication {
    public static void main(String[] args) {
        SpringApplication.run(CacheApplication.class, args);
    }
}
```

通过上述配置和代码，Spring Cache和缓存注解可以在Spring应用中无缝工作。AOP和缓存抽象机制使得这些缓存操作透明地集成到业务逻辑中，大大简化了开发和维护工作。