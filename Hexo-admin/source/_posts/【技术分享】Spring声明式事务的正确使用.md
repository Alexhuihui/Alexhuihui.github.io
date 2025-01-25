---
title: 【技术分享】Spring声明式事务的正确使用
date: 2024-01-18 19:54:06
categories: 技术分享
tags: 技术分享

---

大家肯定对@Transactional这个注解很熟悉，也对事务有着详细的了解，也知道多个数据库操作需要通过事务来保证一致性和原子性。但是很少会关注事务是否生效、有没有出错。这类问题也比较难在测试阶段发现，当出现线上问题的时候不可避免的会产生大量脏数据。所以这次我分享的内容就是帮助大家理清楚使用@Transactional的思路，避免使用不当产生bug。

<!-- more --> 

### 事务为什么不生效

很多同学以为只要加了这个注解就不需要管它了，其实不尽然。可以看看下述例子：

```java
@Service
@Slf4j
public class UserService {

    @Autowired
    private UserRepository userRepository;

    //一个公共方法供Controller调用，内部调用事务性的私有方法
    public int createUserWrong1(String name) {
        try {
            this.createUserPrivate(new UserEntity(name));
        } catch (Exception ex) {
            log.error("create user failed because {}", ex.getMessage());
        }
        return userRepository.findByName(name).size();
    }

    //标记了@Transactional的private方法
    @Transactional
    private void createUserPrivate(UserEntity entity) {
        userRepository.save(entity);
        if (entity.getName().contains("test")) {
            throw new RuntimeException("invalid username!");
        }
    }

    //根据用户名查询用户数
    public int getUserCount(String name) {
        return userRepository.findByName(name).size();
    }
}
```

1. 只有定义在public方法上的 @Transactional 才能生效

```java
public int createUserWrong2(String name) {
    try {
        this.createUserPublic(new UserEntity(name));
    } catch (Exception ex) {
        log.error("create user failed because {}", ex.getMessage());
    }
    return userRepository.findByName(name).size();
}

//标记了@Transactional的public方法
@Transactional
public void createUserPublic(UserEntity entity) {
    userRepository.save(entity);
    if (entity.getName().contains("test")) {
        throw new RuntimeException("invalid username!");
    }
}
```

1. 必须通过代理过的类从外部调用目标方法才能生效

```java
@GetMapping("right2")
public int right2(@RequestParam("name") String name) {
    try {
        userService.createUserPublic(new UserEntity(name));
    } catch (Exception ex) {
        log.error("create user failed because {}", ex.getMessage());
    }
    return userService.getUserCount(name);
}



try {
 // This is an around advice: Invoke the next interceptor in the chain.
 // This will normally result in a target object being invoked.
 retVal = invocation.proceedWithInvocation();
}
catch (Throwable ex) {
 // target invocation exception
 completeTransactionAfterThrowing(txInfo, ex);
 throw ex;
}
finally {
 cleanupTransactionInfo(txInfo);
}
```

1. 只有异常传播出了标记了 @Transactional 注解的方法，事务才能回滚

```java
/**
 * The default behavior is as with EJB: rollback on unchecked exception
 * ({@link RuntimeException}), assuming an unexpected outcome outside of any
 * business rules. Additionally, we also attempt to rollback on {@link Error} which
 * is clearly an unexpected outcome as well. By contrast, a checked exception is
 * considered a business exception and therefore a regular expected outcome of the
 * transactional business method, i.e. a kind of alternative return value which
 * still allows for regular completion of resource operations.
 * <p>This is largely consistent with TransactionTemplate's default behavior,
 * except that TransactionTemplate also rolls back on undeclared checked exceptions
 * (a corner case). For declarative transactions, we expect checked exceptions to be
 * intentionally declared as business exceptions, leading to a commit by default.
 * @see org.springframework.transaction.support.TransactionTemplate#execute
 */
@Override
public boolean rollbackOn(Throwable ex) {
     return (ex instanceof RuntimeException || ex instanceof Error);
}
```

1. 默认出现 RuntimeException（非受检异常）或 Error 的时候，Spring 才会回滚事务（可以指定回滚异常）

### 事务传播

有现在这个场景，注册主会员和注册子用户，要求注册子用户失败不影响主会员的注册，下述伪代码，可以猜一下结果

```java
@Autowired
private UserRepository userRepository;
@Autowired
private SubUserService subUserService;

@Transactional
public void createUserWrong(UserEntity entity) {
    createMainUser(entity);
    subUserService.createSubUserWithExceptionWrong(entity);
}

private void createMainUser(UserEntity entity) {
    userRepository.save(entity);
    log.info("createMainUser finish");
}
```

上述代码异常跑出了@Transactional 注解标记的 createUserWrong 方法，Spring 会回滚事务

修改之后如下：

```java
@Transactional
public void createUserWrong2(UserEntity entity) {
    createMainUser(entity);
    try {
        subUserService.createSubUserWithExceptionWrong(entity);
    } catch (Exception ex) {
        log.error("create sub user error:{}", ex.getMessage());
    }
}
```

虽然捕获了异常，但是因为没有开启新事务，而当前事务因为异常已经被标记为rollback了，所以最终还是会回滚。

看到这里就清楚了，只能让处理子用户的逻辑运行在单独的事务中，这就用到了SPRING的事务传播机制

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void createSubUserWithExceptionRight(UserEntity entity) {
    log.info("createSubUserWithExceptionRight start");
    userRepository.save(entity);
    throw new RuntimeException("invalid status");
}

@Transactional
public void createUserRight(UserEntity entity) {
    createMainUser(entity);
    try {
        subUserService.createSubUserWithExceptionRight(entity);
    } catch (Exception ex) {
        // 捕获异常，防止主方法回滚
        log.error("create sub user error:{}", ex.getMessage());
    }
}
```

希望这次分享能帮助大家正确处理业务代码中的事务。

### 实际使用踩的坑

#### 多数据源中不能使用事务

#### 事务和分布式锁使用的坑

现象：业务中加了分布式锁，也做了幂等处理如果业务ID存在相同的数据则直接返回。但还是会生成2条一样的数据。

```java
@Around("@annotation(redisLock)")
public Object around(ProceedingJoinPoint joinPoint, RedisLock redisLock) throws Throwable {
    String spel = redisLock.key();
    String lockName = redisLock.lockName();
    String redisLockKey = getRedisKey(joinPoint, lockName, spel);
    log.info("生成的 redisKey 是 -> {}", redisLockKey);
    RLock rLock = redissonClient.getLock(redisLockKey);
    Object result;
    try {
        rLock.lock(redisLock.expire(), redisLock.timeUnit());
        //执行方法
        result = joinPoint.proceed();
    } catch (InterruptedException interruptedException) {
        log.error("获取分布式锁失败, ", interruptedException);
        throw new RuntimeException("获取分布式锁失败");
    } finally {
        if (rLock.isLocked()) {
            rLock.unlock();
        }
    }
    return result;
}
@Transactional(rollbackFor = RuntimeException.class)
@RedisLock(lockName = "test", key = "test")
public void test() {
    // 根据手机号查询用户
    User user = bizService.queryByPhone("17770848782");
    log.info("user==={}", user);
}

@Transactional(rollbackFor = RuntimeException.class)
public void test2() {
    // 根据手机号查询用户
    User user = bizService.queryByPhone("17770848782");
    log.info("user2==={}", user);
}
```

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/c87dacaa25d90940960381b89cbcc36.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/c87dacaa25d90940960381b89cbcc36.png)

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/2c863c0d9bede38ff6d4494ffd1e913.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/2c863c0d9bede38ff6d4494ffd1e913.png)

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/31a1043cd83c4866d202ba4c86efd5a.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/31a1043cd83c4866d202ba4c86efd5a.png)

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/152cfd85f8f8c853ed32f308155f461.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/152cfd85f8f8c853ed32f308155f461.png)

```java
@GetMapping
@Operation(summary = "test2")
@RedisLock(lockName = "test2", key = "test2")
public ObjectResponse<String> test2() {
    userAppService.test2();
    return ObjectResponse.success();
}

@PostMapping
@Operation(summary = "test")
public ObjectResponse<String> test() {
    userAppService.test();
    return ObjectResponse.success();
}
```

结论：当使用分布式锁和事务时，因为切面处理是用的先进后出的栈，又因为事务处理的优先级是默认最低的，所以如果没有知道分布式锁的优先级就会导致锁先于事务释放，导致可能出现事务的隔离性被破坏，产生脏数据。