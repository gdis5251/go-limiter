# Go 基于 Redis + Lua 实现分布式限流器

> **限流算法在分布式系统设计中有广泛的应用，特别是在系统的处理能力有限的时候，通过一种有效的手段阻止限制范围外的请求继续对系统造成压力，避免系统被压垮，值得开发工程师们去思考。**

实际生活中，限流器算法通常作为限制用户行为的一种方式之一。比如最近我在某东抢 PS5，开始购买的一瞬间就没了，肯定是有些用户使用了脚本去抢(黑产！)，导致我们用手的人很难抢到。那么限流器就可以限制一下这些通过脚本去抢购的用户，强烈建议某东优化！

## 1. 简单计数限流器

首先要介绍的是一种稍微简单的策略，使用一个计数器，限制用户在指定时间内某个行为最多能发生 N 次。由于现在的系统大多都是使用分布式部署的，所以这个计数器我们使用 Redis 来代替。但是由于存在高并发的问题，如何保障计数器的原子性，也是我们需要讨论的问题。

我们先定义一下限流接口：

```go
// 指定用户uid的某个行为action在特定时间内period能够最多发生maxCount次，其中period单位为秒
func isAllowed(uid string, action string, period, maxCount int) bool {
// 基于redis的简单counter算法
// ...
}

// 下单接口
func CreateOrder() {
	canCreateOrder := isAllowed("berryjam", "createOrder", 5, 50) // 调用限流接口，5秒内最多只能下单50次
	if canCreateOrder {
		// 处理下单逻辑
		// ...
		fmt.Println("下单成功")
	} else { // 返回请求或者抛出异常、panic
		panic("下单次数超限")
	}
}
```

### 解决方案

我们需要维护一个时间滑动窗口，结合 redis 的 zset 数据结构，就可以轻松地通过 score(我们用的时间戳) 来找出指定的时间窗口。每次可以先删除掉时间窗口范围外的值，剩下的值就是这段时间内的值。zset 的 value 为了保证唯一性，我们使用纳秒级时间戳作为 value。

如图 1 所示：

![image-20210811173335334](https://tva1.sinaimg.cn/large/008i3skNly1gtcze7q7kej60e20dtgly02.jpg)

​																**图 1 滑动窗口示意图**

为了减少算法的空间复杂度，只需要保留时间窗口内的行为记录，另外如果用户是不活跃用户，滑动时间窗口内的行为记录也是空的，那么这个zset就可以直接删除，进一步减少占用的空间。

接着通过统计滑动时间窗口内的元素数量与阀值maxCount进行比较，便可知道当前行为是否允许。golang实现代码如下：

```go
package main

import (
	"github.com/go-redis/redis"
	"fmt"
	"time"
)

var redisdb *redis.Client

var counterLuaScript = `
	-- 记录行为
	redis.pcall("zadd", KEYS[1], ARGV[1], ARGV[1]); -- value 和 score 都使用纳秒时间戳，即ARGV[1]
	redis.pcall("zremrangebyscore", KEYS[1], 0, ARGV[2]); -- 移除时间窗口之前的行为记录，剩下的都是时间窗口内的
	local count = redis.pcall("zcard", KEYS[1]); -- 获取窗口内的行为数量
	redis.pcall("expire", KEYS[1], ARGV[3]); -- 设置 zset 过期时间，避免冷用户持续占用内存
	return count -- 返回窗口内行为数量`

var evalSha string

func init() {
	initRedisClient()
}

func initRedisClient() {
	redisdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,
	})

	var err error
	evalSha, err = redisdb.ScriptLoad(counterLuaScript).Result()
	if err != nil {
		panic(err)
	}
}

// period单位为秒
func isAllowed(uid string, action string, period, maxCount int) bool {
	key := fmt.Sprintf("%v_%v", uid, action)
	now := time.Now().UnixNano()
	beforeTime := now - int64(period*1000000000)
	res, err := redisdb.EvalSha(evalSha, []string{key}, now, beforeTime, period).Result()
	if err != nil {
		panic(err)
	}
	if res.(int64) > int64(maxCount) {
		return false
	}
	return true
}

func CreateOrder() {
	canCreateOrder := isAllowed("berryjam", "createOrder", 5, 10)
	if canCreateOrder {
		// 处理下单逻辑
		// ...
		fmt.Println("下单成功")
	} else { // 返回请求或者抛出异常、panic
		panic("下单次数超限")
	}
}

func main() {
	for i := 0; i < 100; i++ {
		CreateOrder()
	}
}
```

isAllowed 接口使用了 lua 脚本来保证 redis 执行的原子性，防止高并发场景下发生无法预料的错误。

整体思路：每当一个行为到来时，都会维护一次时间窗口；将时间窗口外的记录全部清除掉，只保留窗口内的记录。最后获取窗口内记录的 count，与允许的最大值比较。

至此，一个基于 Redis 的限流器已经完成，但是这种方案有一个比较明显的缺点，当时间窗口内的记录特别大时，就需要消耗巨大的内存来维护这个时间窗口。比如 限定 120s 内操作不得超过 1000 万次，就会消耗巨大的资源去维护计数器。

那么就引出下一种优化策略，漏斗限流。

## 2. 漏斗限流

漏斗限流是另一种常见的限流方法，这个算法灵感来源自漏斗(funnel)结构。

如图 2 所示：

![image-20210811175345641](https://tva1.sinaimg.cn/large/008i3skNly1gtczz5mmxqj60d8097dfz02.jpg)

​													**图 2 漏斗算法**

漏斗的容量有限，并且漏嘴的流率固定。如果漏斗停止灌水，那么若干时间后漏斗会变空。如果把漏嘴堵住，一直往里灌水，漏斗会变满直到溢出再也装不进去。如果漏嘴放开，等流走一部分水后，又可以往里面灌水。另外如果漏嘴流水速率大于灌水速率，那么漏斗永远不会满。如果漏嘴流水速率小于灌水速率，那么一旦漏斗满了，就需要暂停灌水等漏斗流出足够的空间后才能继续往里灌水。

因此，漏斗的剩余空间可以表示当前可以进行的行为数量，漏嘴流速表示系统能够处理该行为的频率。下面是用golang实现的一个完整示例代码：

```go
package main

import (
	"time"
	"fmt"
	"github.com/go-redis/redis"
)

const (
	FAILED = iota
	SUCC
)

var redisdb *redis.Client

var initFunnelScript = `
	-- 分别初始化漏斗结构的4个字段capacity、left_quota、leaking_rate、leaking_time
	-- capacity:漏斗容量
	-- left_quota:漏斗剩余空间
	-- leaking_rate:漏嘴流水速率
	-- leaking_time:上一次漏水时间
	local key
	for i,j in ipairs(ARGV) 
	do if i%2 == 0
		then
			redis.pcall('hsetnx', KEYS[1], key, j)
		else
			key = j
		end
	end`

var initFunnelSha string

var makeSpaceScript = `
	local leaking_time = tonumber(redis.pcall('hget', KEYS[1], 'leaking_time'))
	local leaking_rate = tonumber(redis.pcall('hget', KEYS[1], 'leaking_rate'))
	local left_quota = tonumber(redis.pcall('hget', KEYS[1], 'left_quota'))
	local capacity = tonumber(redis.pcall('hget', KEYS[1], 'capacity'))
	local now = tonumber(ARGV[1])
	local delta_time = now - leaking_time -- 距离上一次漏水过去了多久
	local delta_quota = leaking_rate * delta_time -- 又可以腾出不少空间了
	
	redis.pcall('hset', KEYS[1], 'leaking_time', now) -- 记录漏水时间
	if delta_quota + left_quota >= capacity then -- 剩余空间不得高于容量
		redis.pcall('hset', KEYS[1], 'left_quota', capacity) 
	else 
		redis.pcall('hset', KEYS[1], 'left_quota', delta_quota + left_quota) -- 增加剩余空间
	end
`
var makeSpaceSha string

var wateringScript = `
	local left_quota = tonumber(redis.pcall('hget', KEYS[1], 'left_quota'))
	local quota = tonumber(ARGV[1])
	if left_quota >= quota then -- 判断剩余空间是否足够
		redis.pcall('hset', KEYS[1], 'left_quota', left_quota-quota) 
		return 1
	else
		return 0
	end
`

var wateringSha string

func init() {
	initRedisClient()
}

func initRedisClient() {
	redisdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,
	})

	var err error
	initFunnelSha, err = redisdb.ScriptLoad(initFunnelScript).Result()
	if err != nil {
		panic(err)
	}

	makeSpaceSha, err = redisdb.ScriptLoad(makeSpaceScript).Result()
	if err != nil {
		panic(err)
	}

	wateringSha, err = redisdb.ScriptLoad(wateringScript).Result()
	if err != nil {
		panic(err)
	}
}

func MakeSpace(key string) {
	now := time.Now().Unix()
	redisdb.EvalSha(makeSpaceSha, []string{key}, now).Result()
}

// quota为每次处理请求所需要的资源配额
func Watering(key string, quota float64) bool {
	MakeSpace(key)
	res, err := redisdb.EvalSha(wateringSha, []string{key}, quota).Result()
	if err != nil {
		panic(err)
	}
	return res.(int64) == SUCC
}

func IsActionAllowed(uid, action string, capacity float64, leakingRate float64) bool {
	key := fmt.Sprintf("%v_%v", uid, action)
	redisdb.EvalSha(initFunnelSha, []string{key}, "capacity", capacity, "left_quota", capacity, "leaking_rate", leakingRate, "leaking_time", time.Now().Unix())
	return Watering(key, 1)
}

func main() {
	for i := 0; i < 20; i++ {
		fmt.Printf("%+v\n", IsActionAllowed("berryjam", "reply", 15, 0.5))
	}
}
```

整体思路：

每次先初始化该动作的 redis 结构，如果 hsetnx 已经存在，则操作不会生效。

然后根据初始化的 QPS(leakingRate)，和leaking_time(上一次漏水的时间)可以计算出现在还剩下多少空间。

最后如果空间允许的情况下，对空间减少 n 个值。

> Redis4.0本身提供了一个限流redis模块，叫redis-cell[3]。该模块也使用了漏斗算法，并提供了原子的限流指令，限流问题会变得更加简单。

叮~:bell:


