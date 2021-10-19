# Redis

常用的数据结构

`String, list, hash, set, zset(sorted set)`

数据结构如何使用，用在哪些场景

除了上述类型，还有什么其它 redis 类型

```code
bitmap(位图)

HyperLogLogs(统计)

geoSpatial(地理信息)

Stream
```

命令不区分大小写，键值 `key` 区分大小写

`help @ String help` 命令可以直接查看某种类型数据

## String

String:

```shell
SET key value
MSET k1 v1 k2 v2 k3 v3  ==> m: more
MGET key k[key...]

INCR key
INCEBY
DECR
DECRBY

STRLEN key

应用在分布式锁中

SETNX key value  => set if not exists 创建，如果不存在的花
SET key value [EX seconds][PX milliseconds]{NX|XX}

上边第二条命令的含义
EX: key 在多少秒之后过期
PX: key 在多少毫秒后过期
NX: key 不存在的时候，才创建 key, 效果等同于 setnx
XX: key 存在的时候，覆盖 key
```

应用：商品编号、订单号采用 `INCR` 生成，喜欢数，阅读数量，点击 + 1

## Hash

hash:

redis 中的 `hash` 对应 Java 的 `map<string, map<k, v>>`

```shell
HSET key field value  -- 一次设置一个字段值
HGET key field -- 一次获取一个字段值
HMSET key field value [filed value ...] -- 一次设置多个字段值
HMGET key field [field ...] 一次获取多个字段值
HGETALL key -- 获取所有字段值
HLEN -- 获取某个 key 内的全部数量
HDEL -- 删除一个 key
```

应用：购物车早期，中小厂可以使用

## List

list:
有序，有重复，有序

```shell
LPUSH key value [value ...] -- 向列表左边添加元素
RPUSH key value [value ...] -- 向列表右边添加元素
LRANGE key start stop -- 查看列表
LLEN key -- 获取列表中元素个数
```

应用场景：

一个人对应的微信公众号 `1:N`

## Set

set:
无序，无重复

```shell
SADD key member [member ...] -- 添加元素
SREM key member [member ...] -- 删除元素
SMEMBERS key -- 获取集合中的所有元素
SISMEMBER key member -- 判断元素是否在集合中
SCARD key -- 获取集合中元素个数
SRANDMEMBER key [数字] -- 从集合中随机弹出一个元素，元素不删除. [数字] 表示弹出几个
SPOP key [数字] -- 从集合中随机弹出一个元素，出一个删一个
```

集合运算：

```shell
差集运算 A-B
SDIFF key [key ...]
交集运算 A&&B
SINTER key [key ...]
并集运算 A || B
SUNION key [key ...]
```

应用：
微信的抽奖小程序：随机弹出一个/多个数字, 使用 `SPOP` 删除的， 立即参与相当于执行 `SADD`, `SCARD` 统计总数
微信朋友圈点赞：`SADD` 点赞，`SREM` 取消点赞, 点赞用户数统计 `SCARD`, 判断用户是否点赞过 `SMEMBERS`, 展示所有的 `SMEMBERS`
微薄的好友关注关系: 交集，并集，差集等等，共同关注, 我关注的人也关注他
QQ 可能认识的人

## Zset

有序的集合中加入一个元素和这个元素的分数

ZADD key score member [socre member]

zset 应用

根据商品的销售对商品进行排序显示
抖音热搜

## 分布式锁

你知道分布式锁吗？有哪些实现方案？redis 分布式锁的理解，删 key 的时候有什么问题

JVM 层面的加锁

分布式微服务架构，拆分后各个微服务之间为了避免冲突数据故障而加入的一种锁，分布式锁

三种分布式锁

1. mysql
2. zookeeper
3. redis

现在习惯用 redis 做分布式锁

redis -> redlock -> redisson lock/unlock 方法

Redlock 是一种理念，redisson 是一种落地实现

自己手写需要 setnx + lua 脚本首先

RedisCluster -> redisson

为什么使用 redisson？

1. 单机版没有加锁，增加 synchronized/ReentrantLock，两者区别，不见不散/过时不候， synchronized 导致线程积压，线程大量等待/放弃等待，拿不到锁规定时间内放弃，ReentrantLock 是有条件的等待，小规模的等待 lock.tryLock()
2. 分布式部署后，单机锁还是出现超卖现象，需要分布式锁，因为单机版每一个服务中的锁都不一样，加分布式锁 SETNX
3. 去掉 synchronized 然后添加分布式锁

```java
String value = UUID.randomUUID().toString() + Thread.currnetThread().getName;
Boolean flag = stringRedisTemplate.opsForValue().setyIfAbsent(REDIS_LOCK, value);
if(!flag){
    return "抢锁失败！"
}else{
    // 正常业务
}

// 解锁, 删除该 Lock
stringRedisTemplate.delete(REDIS_LOCK);
```

1. 程序如果出现异常，那么锁一定要被释放掉，必须在代码层面加一个 finally()
2. lock/unlock() 必须同时出现

- 宕机了，程序被干掉了，程序无法走到 finally 代码中，没办法保证解锁，key 没有被删除，需要加入一个过期时间自动删除解锁

```java
Boolean flag = stringRedisTemplate.opsForValue().setyIfAbsent(REDIS_LOCK, value);
// 设置过期时间
stringRedisTemplate.expire(10L, TimeUnit.SECONDS);
```

- 设置 key 和过期时间分开，二者操作不是原子，需要同时操作

```java
Boolean flag = stringRedisTemplate.opsForValue().setyIfAbsent(REDIS_LOCK, value, 10L, TImeUnit.SECONDS);
```

- 过期时间太短，导致锁提前被释放，线程 A 在规定时间内没执行完，然后执行完之后，将其它线程创建的锁删除，所以需要降低锁的粒度

```java
//判断锁时自己的锁，才去删除
if(= stringRedisTemplate.opsForValue().get(REDIS_LOC).equals(value)){
    // 才去删除
}
```

判断加锁和解锁并不是同一个客户端，可能会造成误解锁
finally 块的判断值 + del 删除不是原子操作，会导致吴解锁

采用 lua 脚本，进行删除，lua 脚本为什么可以保证原子性？

***Redis 脚本为什么能够保证原子性***

Redis 使用单个 Lua 解释器运行所有脚本，并且，Redis 也保证脚本会以原子性(atomic) 的方式执行：

具体来说，当某个脚本正在运行的时候，不会有其它脚本活 Redis 命令被执行。
这和使用 `MULTI/EXEC` 包围的事务类似，在其它客户端看来，脚本的效果要么是可见的，要么就是已经完成的

尽管脚本的运行开销非常少，但是也需要注意脚本的效率，如果脚本执行很慢的花，会阻塞其它的服务器

```lua
-- lua删除锁：
-- KEYS和ARGV分别是以集合方式传入的参数，对应上文的Test和uuid。
-- 如果对应的value等于传入的uuid。
if redis.call('get', KEYS[1]) == ARGV[1] 
    then 
 -- 执行删除操作
        return redis.call('del', KEYS[1]) 
    else 
 -- 不成功，返回0
        return 0 
end
```

业务代码：

```java
finally{
    Jedis jedis = RedisUtils.getJedis();

    String script = "if redis.call('get', KEYS[1]) == ARGV[1] \n" +
            "    then \n" +
            " -- 执行删除操作\n" +
            "        return redis.call('del', KEYS[1]) \n" +
            "    else \n" +
            " -- 不成功，返回0\n" +
            "        return 0 \n" +
            "end";
    
    try{
        Object o = jedis.eval(script, Collections.singletonList(REDIS_LOCK), Collections.singletonList(value))
        if("1".equals(O.toString())){
            System.out.println("del redis lock ok");
        }else{
            System.out.println("del redis lock fail");
        }
    }finally{
        if(null != jedis){
            jedis.close();
        }

    }
}

```

如果不用 lua 脚本，还可以采用什么方案？
采用 redis 自身的事务

Redis 事务！

```java
while(true){
    stringRedisTemplate.watch(REDIS_LOCK);
    //判断锁时自己的锁，才去删除
    if(= stringRedisTemplate.opsForValue().get(REDIS_LOC).equals(value)){
        // 才去删除
        stringRedisTemplate.setEnableTransactonSupport(true);
        stringRedisTemplate.multi();
        stringRedisTemplate.delete(REDIS_LOCK);
        List<Object> list = stringRedisTemplate.exec();
        if(list == null){
            contunue;
        }

    }
    stringRedisTemplate.unwatch();
    break;
}
```

- 确保 redisLock 过期时间大于业务执行时间的问题， Redis 分布式锁如何现缓存续期？

如何保证 Redis 锁超时时间内任务一定完成，加时间不一定能够实现。不知道业务逻辑到底执行多长时间！

- Redis 目前是单机版，总从，集群 + CAP + Zoopkeeper 实现

Redis 保证 AP 高可用性和分区容错性

Redis 异步赋值造成丢失，主节点没来得及把刚刚 set 进来的这条数据给从节点点，就挂掉了，从节点上位变成主节点，但是新的主节点中没有 set 进来的键值，牺牲了数据一致性

Zookeeper CP，先不着急恢复，先保证从节点数据保持一致，从节点数据和主节点数据保持一致

- 所以直接使用 RedLock 之 Redisson 落地实现

```java
@Autowired
private Redisson redisson

RLock redissonLock = redisson.getLock(REDIS_LOCK);

redissonLock.lock();

try{

}finally{
    redisson.unlock();
}
```

- 结尾完善, fianally 中的 unlock() 方法不要直接写，可能偶尔会有错误，超高并发， attempt to unlock lock, not lock by current thread by node id:

```java
if(redissonLock.isLocked()){
    if(redissonLock.isHeldByCurrentThread()){
        redissonLock.unlock();
    }
}
```

todo
为什么会出现这样的错误？？

续命锁如何实现

WathchDow 如何实现

### redis 内存调整默认策略

生产上 redis 内存设置多少，配置，修改 redis 的内存大小

```shell
# 查看 redis 最大占用内存
# 配置文件中：
maxmemory
# 如果不设置最大内存，64 位不设置最大内存，32位最大的内存 3GB
# 单位是 <bytes>

# 生产上的配置
一般设置为最大物理内存的 3/4

# 如何设置 redis 内存
1. 修改配置文件
maxmemory 14857600 # 100M
2. 通过命令修改

shutdown
redis-server /myredis/redis.conf
redis-cli
# config 命令
config get maxmemory
config set maxmemory 1

# 查看 redis 内存使用情况
info memory

查看的是 redis 内存的各种情况

```

内存满了怎么办？

设置为 1 byte, 直接报 (error) OOM command not allowed when used memory > 'maxmemory'

redis 清理内存的方式，定期删除和惰性删除了解过吗

redis 缓存淘汰策略

redis 的 LRU 了解过吗？手写 LRU 算法
