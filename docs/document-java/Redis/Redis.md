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
2. 