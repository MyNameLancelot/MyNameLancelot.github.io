---
layout: post
title: "分布式锁"
date: 2022-02-15 19:37:37
categories: Tool
keywords: "分布式锁,分布式锁的使用,redission,lock"
description: "分布式锁,分布式锁的使用,redission,lock"
---

## 一、分布式锁起因

**分布式锁出现的原因**

​	在传统单体应用单机部署的情况下，可以使用并发处相关的功能（如java并发处理相关的API：ReentrantLock或者syncchronized等）进行互斥控制来解决。但随着业务发展，系统架构也会逐步优化升级，原本单体单机部署的系统被演化为分布式集群系统。由于分布式系统多线程，多进程并且分布在多个不同机器上，这将使原来单机部署情况下的并发控制锁策略无法满足，并不能提供分布式锁的能力。为了解决这个问题就需一个跨机器的互斥机制来控制共享资源的访问，这就是分布式锁的解决的难题。

---

**分布式锁应用的场景**

- 提升效率：如果不使用分布式锁，会导致业务重复执行一些没有意义的工作
- 正确性： 使用分布式锁可以防止对数据的并发访问，避免数据不一致，数据损失等

---

**分布式锁需要具备的特性**

| 特性     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 排他性   | 同一时间只会有一个客户端能获取到锁，其它客户端无法同时获取   |
| 避免死锁 | 锁在一段时间内有效，超过这个时间后会被释放（正常释放或异常释放） |
| 高可用   | 获取或释放锁的机制必须高可用且性能不能过差                   |

## 二、使用Redission实现分布式锁

### Redis锁自实现及其问题

首先我们可以使用Redis实现初步具有锁能力的代码

```java
@Test
public void testDistributedLockRedis() {
  String LOCK_KEY = "goods_001";
  String lockThreadFlag = UUID.randomUUID().toString();
  Boolean lockResult = stringRedisTemplate.opsForValue().setIfAbsent(LOCK_KEY, lockThreadFlag, 30, TimeUnit.SECONDS);
  try {
    if (Boolean.TRUE.equals(lockResult)) {
      // 执行业务代码
      doBusinessCode();
      // ....
    }
  } finally {
    if (Boolean.TRUE.equals(lockResult) && lockThreadFlag.equals(stringRedisTemplate.opsForValue().get(LOCK_KEY))) {
      stringRedisTemplate.delete(LOCK_KEY);
    }
  }
}
```

<span style="color:red">问题：①如果业务代码执行的时间超过锁过期时间那么资源锁被释放了，就会有并发问题。如果时间设置过久，程序宕机没有释放锁，会导致锁时间过长。②重入问题没有考虑</span>

<span style="color:blue">为解决以上问题，需要获得锁的线程开启一个守护线程，用来给快要过期的锁"续航"。</span>例如每过10s检查，如果业务代码没执行完则重设锁时长为30。由于业务线程和守护线程在同一个进程，业务线程执行完成或者终止，守护线程也会停下。这把锁到了超时的时候，没人给它续命，也就自动释放了。但是编写这些代码在实际生产过程种可能要考虑更多问题，此时我们可以用`Redisson`框架的封装完善的锁。

---

### Redission锁的原理

redission加锁和解锁采用的是Lua脚本，需要先研究一下加锁和解锁脚本都做了什么

**加锁脚本**

```lua
-- 若锁不存在：则新增锁，并设置锁重入计数为1、设置锁过期时间
if (redis.call('exists', KEYS[1]) == 0) then
  redis.call('hset', KEYS[1], ARGV[2], 1);
  redis.call('pexpire', KEYS[1], ARGV[1]);
  return nil;
end;

-- 若锁存在，且唯一标识也匹配：则表明当前加锁请求为锁重入请求，故锁重入计数+1，并再次设置锁过期时间
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
  redis.call('hincrby', KEYS[1], ARGV[2], 1);
  redis.call('pexpire', KEYS[1], ARGV[1]);
  return nil;
end;

-- 若锁存在，但唯一标识不匹配：表明锁是被其他线程占用，当前线程无权解他人的锁，直接返回锁剩余过期时间
return redis.call('pttl', KEYS[1]);
```

加锁脚本流程解读

<img src="C:/img/distributed-lock/lock_script.png" alt="lock_script" style="zoom: 50%;" />

**解锁脚本**

```lua
-- 若锁不存在：则直接广播解锁消息，并返回1
if (redis.call('exists', KEYS[1]) == 0) then
  redis.call('publish', KEYS[2], ARGV[1]);
  return 1; 
end;

-- 若锁存在，但唯一标识不匹配：则表明锁被其他线程占用，当前线程不允许解锁其他线程持有的锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
  return nil;
end; 

-- 若锁存在，且唯一标识匹配：则先将锁重入计数减1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 
if (counter > 0) then 
  -- 锁重入计数减1后还大于0：表明当前线程持有的锁还有重入，不能进行锁删除操作，但可以友好地帮忙设置下过期时期
  redis.call('pexpire', KEYS[1], ARGV[2]); 
  return 0; 
else 
  -- 锁重入计数已为0：间接表明锁已释放了。直接删除掉锁，并广播解锁消息，去唤醒那些争抢过锁但还处于阻塞中的线程
  redis.call('del', KEYS[1]); 
  redis.call('publish', KEYS[2], ARGV[1]); 
  return 1;
end;

return nil;
```

解锁脚本流程解读

<img src="C:/img/distributed-lock/unlock_script.png" alt="unlock_script" style="zoom:50%;" />

广播解锁消息的作用：通知其它争抢锁阻塞住的线程，从阻塞中解除，并再次去争抢锁

**加锁和解锁总流程图**

![lock_unlock_flower](C:/img/distributed-lock/lock_unlock_flower.png)

### Redission的使用

-  普通非公平重入锁

```java
/**
  * 普通非公平重入锁
  */
@Test
public void testRedissionLock() {
  String LOCK_KEY = "goods_001";
  //获取分布式锁，只要锁的名字一样，就是同一把锁
  RLock lock = redissonClient.getLock(LOCK_KEY);
  //加锁（阻塞等待），默认过期时间是30秒，实现的JDK的LOCK接口，也可使用tryLock尝试加锁
  lock.lock();
  try {
    //如果业务执行过长，Redisson会自动给锁续期
    doBusinessCode();
  } finally {
    //解锁，如果业务执行完成，就不会继续续期，即使没有手动释放锁，在30秒过后，也会释放锁
    lock.unlock();
  }
}
```

- 公平锁

```java
/**
  * 公平锁
  */
@Test
public void testRedissionFairLock() {
  String LOCK_KEY = "goods_001";
  //获取分布式锁，只要锁的名字一样，就是同一把锁
  RLock lock = redissonClient.getFairLock(LOCK_KEY);
  //加锁（阻塞等待）
  lock.lock();
  try {
    //如果业务执行过长，Redisson会自动给锁续期
    doBusinessCode();
  } finally {
    //解锁，如果业务执行完成，就不会继续续期，即使没有手动释放锁，在30秒过后，也会释放锁
    lock.unlock();
  }
}
```

- 读写锁

```java
/**
  * 读写锁
  */
@Test
public void testRedissionReadWriteLock() {
  String LOCK_KEY = "goods_001";
  //获取分布式锁，只要锁的名字一样，就是同一把锁
  RReadWriteLock lock = redissonClient.getReadWriteLock(LOCK_KEY);
  //加锁（阻塞等待）
  RLock readLock = lock.readLock();
  try {
    //如果业务执行过长，Redisson会自动给锁续期
    doBusinessCode();
  } finally {
    //解锁，如果业务执行完成，就不会继续续期，即使没有手动释放锁，在30秒过后，也会释放锁
    readLock.unlock();
  }
}
```

- 批量连锁

```java
/**
  * 批量连锁
  */
@Test
public void testRedissionMultiLock() {
  RLock lock1 = redissonClient.getLock("goods_001");
  RLock lock2 = redissonClient.getLock("goods_002");
  RLock lock3 = redissonClient.getLock("goods_003");
  // 同时对几个资源一起加锁
  RLock lock = redissonClient.getMultiLock(lock1, lock2, lock3);
  //加锁（阻塞等待）
  lock.lock();
  try {
    //如果业务执行过长，Redisson会自动给锁续期
    doBusinessCode();
  } finally {
    //解锁，如果业务执行完成，就不会继续续期，即使没有手动释放锁，在30秒过后，也会释放锁
    lock.unlock();
  }
}
```

- CountDownLatch，和JDK的CountDownLatch用法相同

```java
/**
  * CountDownLatch
  */
@Test
public void testRedissionCountDownLatch() {
  String LOCK_KEY = "TEST_COUNT_DOWN_LATCH";
  // 获取countDownLatch，其它地方有个设置countDown的数量 countDownLatch.trySetCount(10);
  RCountDownLatch countDownLatch = redissonClient.getCountDownLatch(LOCK_KEY);
  countDownLatch.countDown();
  doBusinessCode();
}
```

- Semaphore，和JDK的Semaphore用法相同

```java
/**
  * Semaphore
  */
@Test
public void testRedissionSemaphoreKey() throws InterruptedException {
  String SEMAPHORE_KEY = "TEST_SEMAPHORE";
  RSemaphore semaphore = redissonClient.getSemaphore(SEMAPHORE_KEY);
  semaphore.acquire(2);
  doBusinessCode();
  semaphore.release(2);
}
```

- RedLock

因为Redis是AP架构，主从之间是异步复制。极端情况下如果master节点挂掉，但是slave节点还未同步到master数据，这时候锁会失效。为了避免这种极端情况可以使用RedLock【其实不推荐使用，RedLock本身也存在一些问题，达到CP的效果不如直接使用zookeeper或者etcd这种本就是CP的架构】

算法大概逻辑：部署多台与master节点同等级别的其他节点，这几个Redis不参与其他的业务。每一个线程在向master节点请求锁的同时，也向这几个同等级别的节点发送加锁请求，只有当超过一半的节点数加锁成功，此时的分布式锁才算真正的成功。

缺点：

1. 增加了部署成本，因为使用Redlock需要增加几台与master同等级的节点来实现加锁。这几个节点啥也不干，就只是负责加锁和释放锁逻辑。、
2. 安全争议。让我们假设客户端从大多数Redis实例取到了锁。所有的实例都包含同样的key，并且key的有效时间也一样。然而，key肯定是在不同的时间被设置上的，所以key的失效时间也不是精确的相同。如果客户端在获取到大多数redis实例锁，使用的时间接近或者已经大于失效时间，客户端将认为锁是失效的锁，并且将释放掉已经获取到的锁，所以我们只需要在有效时间范围内获取到大部分锁这种情况。
3. 系统活性争议。系统的活性安全基于三个主要特性: 锁的自动释放（因为key失效了）：最终锁可以再次被使用。客户端通常会将没有获取到的锁删除，或者锁被取到后，使用完后，客户端会主动（提前）释放锁，而不是等到锁失效另外的客户端才能取到锁。当客户端重试获取锁时，需要等待一段时间，这个时间必须大于从大多数Redis实例成功获取锁使用的时间，以最大限度地避免脑裂。然而，当网络出现问题时系统在失效时间(TTL){.highlighter-rouge}内就无法服务，这种情况下我们的程序就会为此付出代价。如果网络持续的有问题，可能就会出现死循环了。这种情况发生在当客户端刚取到一个锁还没有来得及释放锁就被网络隔离。如果网络一直没有恢复，这个算法会导致系统不可用。

## 三、使用Zookeeper实现分布式锁

### 利用Zookeeper实现分布式锁原理

​	zookeeper实现分布式锁的原理就是多个节点同时在一个指定的节点下面创建<span style="color:red">临时会话顺序节点</span>，谁创建的节点序号最小，谁就获得了锁。并且其他节点就会监听序号比自己小的节点【利用zookeeper的Watcher机制】，一旦序号比自己小的节点被删除了，其他节点就会得到相应的事件，然后查看自己是否为序号最小的节点，如果是，则获取锁。

<img src="C:/img/distributed-lock/zookeeper-lock-principle.png" alt="zookeeper-lock-principle" style="zoom:50%;" />

可重入是利用JDK线程ThreadId是否相同判断的

```java
public class ZkLock implements Lock {

  //......
  final AtomicInteger lockCount = new AtomicInteger(0);
  //......

  @Override
  public boolean lock() {
    //可重入，确保同一线程，可以重复加锁
    synchronized (this) {
      if (lockCount.get() == 0) {
        thread = Thread.currentThread();
        lockCount.incrementAndGet();
      } else {
        if (!thread.equals(Thread.currentThread())) {
          return false;
        }
        lockCount.incrementAndGet();
        return true;
      }
      //......
    }
  }
}
```

### 使用Curator完成zookeeper分布式锁

​	Curator是Netflix公司开源的一套zookeeper客户端框架，解决了很多Zookeeper客户端非常底层的细节开发工作，包括连接重连、反复注册Watcher和分布式锁等。

使用Spring Boot初始化Curator的前置代码

```java
@Configuration
public class CuratorConfig {
  @Bean(destroyMethod = "close", initMethod = "start")
  public CuratorFramework curatorFramework() {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    return CuratorFrameworkFactory.builder()
      .connectString("192.168.153.133:2181")
      .sessionTimeoutMs(5000)
      .connectionTimeoutMs(5000)
      .namespace("zookeeper-lock")
      .retryPolicy(retryPolicy)
      .build();
  }
}
```

- 分布式可重入排它锁

```java
@Test
public void testZkMutex() throws Exception {
  String LOCK_KEY = "goods_001";
  InterProcessMutex zkMutex = new InterProcessMutex(curatorFramework, "/"+ LOCK_KEY);
  // 阻塞死等
  zkMutex.acquire();
  try {
    doBusinessCode();
  } finally {
    zkMutex.release();
  }
}
```

- 分布式读写锁

```java
@Test
public void testZkReadLock() throws Exception {
  String LOCK_KEY = "goods_001";
  InterProcessReadWriteLock readWriteLock = new InterProcessReadWriteLock(curatorFramework, "/" + LOCK_KEY);
  // 阻塞死等
  InterProcessMutex readLock = readWriteLock.readLock();
  try {
    doBusinessCode();
  } finally {
    readLock.release();
  }
}
```

- 批量连锁

```java
@Test
public void testZkMultiLock() throws Exception {
  final InterProcessLock lock1 = new InterProcessMutex(curatorFramework, "/lock_good01");
  final InterProcessLock lock2 = new InterProcessMutex(curatorFramework, "/lock_good02");
  InterProcessMultiLock interProcessMultiLock = new InterProcessMultiLock(Arrays.asList(lock1, lock2));
  // 阻塞死等
  interProcessMultiLock.acquire();
  try {
    doBusinessCode();
  } finally {
    interProcessMultiLock.release();
  }
}
```

- Semaphore

```java
@Test
public void testZkSemaphore() throws Exception {
  InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(curatorFramework, "/test_semaphore", 10);
  // 阻塞死等
  Collection<Lease> acquireLease = semaphore.acquire(8);
  try {
    doBusinessCode();
  } finally {
    semaphore.returnAll(acquireLease);
  }
}
```



