# Redis实现分布式锁

> 为了防止分布式系统中的多个进程之间相互干扰，我们需要一种分布式协调技术来对这些进程进行调度。而这个分布式协调技术的核心就是来实现这个分布式锁。

### 实现原理

​		利用 Redis 的 `setnx` 命令。此命令同样是原子性操作，只有在 `key` 不存在的情况下，才能 `set` 成功。加锁是调用 Redis 的 `setnx` 命令，执行成功则返回 1，执行失败返回 0，此时只需要判断返回值就能得知加锁是否成功。释放锁是调用 Redis 的 `del` 命令，删除加锁是 set 的 key，删除成功返回 1，删除失败返回 0。

### 加锁过程

​		当使用 `setnx` 进行加锁且未设置过期时间时，如果某一个线程 A 拿到了这个锁，但是在执行的时候发生了异常（停电了）最终没有执行释放锁的过程，这样其他的线程就拿不到这个锁。

​		当使用 `setnx` 进行加锁且设置过期时间时，如果某个线程 A 执行的时候耗费了很长的时间，超过了锁过期的时间，这样当它执行释放锁的命令 `del` 时，可能没有删除任何 key，这不会产生任何影响。当时当 A 获得的锁超时时，另一个线程 B 在这个时候获得了锁，而线程 A 也执行完了，这样一来线程 B 加的锁就会被线程 A 给删除，此时就发生了**误删**的情况。

​		当出现上述情况的时候，就需要人工介入处理。因此要认真选择锁的超时时间。以下时使用 Jedis 实现的加锁过程。

```java
/**
* @param id 用于标识是谁执行了加锁这个操作
* @param timeout 加锁的超时时间，超过这个时间而且还没有获得锁就会直接返回
* @return 获得锁返回 true，超时返回false
*/
public boolean lock(int id, long timeout) {
    // 获取 Jedis 对象
    Jedis jedis = RedisUtil.getJedis();
    // 获取开始加锁的时间
    long startTime = System.currentTimeMillis();
    // 循环尝试获取锁
    for (; ; ) {
        // SET [key] [id] EX MAX_LOCK_TIME NX
        // 如果执行成功返回 OK，则认为加锁成功，返回 true
        if ("OK".equals(jedis.set(key, String.valueOf(id), setParams))) {
            jedis.close();
            return true;
        } else {
            // 判断是否超时
            if (System.currentTimeMillis() > startTime + timeout) {
                return false;
            }
        }
        try {
            // 10ms 后重试
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 释放锁过程

​		为了避免出现上述误删的情况，在进行释放锁即执行 `del` 命令时需要现取出 key 的 value 值，进行比较，如果时当前线程加的锁才能删除。在 Redis 中，`del` 操作和 `set` 操作都是原子性的，但是取出 value 值并在内存中机械能比较这个过程就不是原子操作了。为了保证原子性，避免代码复杂，就是用 Lua 脚本来实现这个过程。

```java
/**
* @param id 用于标识是谁执行了加锁这个操作
* @return 释放锁成功返回 true，失败返回 false
*/
public boolean unlock(int id) {
    // 获取 Jedis 对象
    Jedis jedis = RedisUtil.getJedis();
    // Lua 脚本
    String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end";
    try {
        // 执行 Lua 脚本
        Object result = jedis
                .eval(script, Collections.singletonList(key), Collections.singletonList(String.valueOf(id)));
        // 返回 1 表示删除成功，0 表示删除失败
        if ("1".equals(result.toString())) {
            return true;
        }
        return false;
    } finally {
        jedis.close();
    }
}
```

### 总结

​		使用 Redis 实现分布式锁，总体上还是可以满足一般的需求，但是还是有很多不足的。

​		首先，对于锁超时，如果线程执行时间过长，锁就会因为超时被删除，这样这个线程在此后的执行过程中的正确性就不能被保证。

​		此外，对于单节点的 Redis 服务器来说，不会产生重复加锁。但是在  redis sentinel 集群中，指令首先在主库执行，然后从库再向主库同步。再线程 A 执行了一次加锁操作 `SET [key] [id] EX MAX_LOCK_TIME NX` ，主库执行完毕，然后就挂掉了，从库没来得及同步。这是就会从剩下的从库中选举出一个节点作为从库。此时称为主库的那个从库并没有那个锁对应的记录。这样如果线程 B 来了获取锁时是可以获取到的。

### 完整代码

```
github https://github.com/Balababaa/redis
```

