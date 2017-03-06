---
layout: post
title: 基于redis的时序ID生成器 | sequence ID base on redis
posttitle: 基于redis的时序ID生成器
date: 2017-02-10 18:30
comments: true
keywords: sequence id generator ID生成器 时序
description: 基于redis的一个生成全局唯一时序的ID的轻量、高效基础架构组件.
external-url:
categories: ID生成器
---

> 随着数据量的增加，采用分库分表后，全局唯一ID生成器是大多数互联网公司都需要的基础架构组件，采用redis的高性能特性，加上时序id算法，可以完美解决唯一ID的问题，本文主要讲其实现算法.

<b>项目使用的源代码[github]</b>: *<a target="blank" href="https://github.com/nice-opentech/id-generator-base-on-redis">id-generator-base-on-redis</a>*

## 目录
1. [为什么需要ID生成器](#idx-why-need-id-generator)
2. [ID生成器的特点](#idx-id-generator-features)
3. [ID生成器如何保证有序](#idx-how-id-generator-sequenced)
4. [ID生成器系统架构选择](#idx-id-generator-system-arch-select)
5. [毫秒内超过shard-range请求次数时，采用借毫秒的方式超前分配ID](#idx-exceed-shard-range-use-borrow-ms)
6. [借毫秒分配ID算法](#idx-borrow-ms-algorithm)
7. [关键代码实现](#idx-id-generator-key-code)
8. [Redis命令支持](#idx-id-generator-redis-command)
9. [压测](#idx-id-generator-press-test)
10. [ID生成器服务部署](#idx-id-generator-service-deploy)
11. [issues](#idx-id-generator-issues)



<a id="idx-why-need-id-generator" />
<hr/>
## 1.为什么需要ID生成器

现如今，大多数互联网公司的数据都相当大，一个持久化的db根本没法存储或者单库单表根本不符合高性能，高吞吐量的标准，因此都采取了分库分表的逻辑来提高读写性能。但分库分表后，主键就不能唯一表示一个
实体了，因为每个表有自己的主键，这就要求有一个全局的主键来确定单一实体，因此需要一个全局的、中心化的、高性能的ID生成器。

<a id="idx-id-generator-features" />
## 2.ID生成器的特点
    
2.1. 高性能、高可用、高并发访问
        
> 分库分表后，写的压力其实没有变，所有的单表都需要同时去拿一个ID生成器的唯一ID，而随着业务的增多，分库分表将成为每个业务的常态，这就对ID生成器提出高性能、高可用、高并发访问的要求。

2.2. 对于所有实体的ID来说，数字是最好的选择, 使用64位整形作为唯一ID

> 分库分表前，大家的业务开发都是采取单表的自增ID实现了主键的逻辑，在业务中都有类似getByID的方法获取实体信息，分库分表后，
  为了更好的的兼容迁移逻辑，业务上更多的无感知，尤其是已发版的(Android|iOS)客户端，采用数字最为合理；另外，随着数据的增多，
  32位数能表示的范围【(2^31 - 1) = 2147483647】可能不足以满足互联网的快速数据增长的需求，因此采用64未长整型比较合适; 

2.3. ID必须是时序的
        
> 业务上很多地方，对id可能有排序的要求，或者最新的列表等，保证id是有序的，减少业务的逻辑错误，这请参考 3. ID生成器如何保证有序.
        
<a id="idx-how-id-generator-sequenced"/>
## 3. ID生成器如何保证有序

3.1 利用时间+有序数保证

ID由64位数字组成，由于数字有一个符号位，因此可用的应该是63位，采用毫秒时间戳加上和有序数组成， 如下结构:
<img src="/image/posts/2017-03-06-64bit-long-int-unique-id-structure.jpeg" />
具体解释如下:

* 符号位: 1bit

* 毫秒时间戳: microtime(): 42bit

>    为了保证递增，采取毫秒时间戳，这样至少在单位毫秒内，数字是递增的，同时另外的福利是，这样的设计，server将是无状态的，所以即使重启也不会影响数据.

* ID生成器server标识: server-id: 2bit

>    为了高可用，一个server很难确保不会挂掉，因此采用多机集群部署，为了防止各个server产生的ID相同，因此每个server给定一个标识，该数字可以分配多位，
    分配两位即可以部署4台机器，一位即可以部署两台机器，这样保证了每个server生成的id不一样，且实现了高可用, 注意: server-id会让毫秒内的id从多个server拿出来的
    id顺序不一致，如果并发不是那么高，并对顺序要求很严格，可以只部署一台

* 毫秒内不重复循环数: shard-range: 9bit

>    该数字是一个毫秒内可以出的数字的总数，2^9 = 512个数，这是循环读取的，保证一个毫秒内不会取到相同的数字，关于一个毫秒内超过了shard-range的上限怎么办？
    请参考：[毫秒内超过shard-range请求次数时，采用借毫秒的方式超前分配id](#idx-exceed-shard-range-use-borrow-ms)

* 随机数: random_range: 10bit
>    这是在使用中遇到的问题，id读取后，可能出现不均匀的情况，因为大部分分表都是按照唯一ID取模分表，比如模1024，如果一旦产生的id不均匀，这会导致某些分表很大，这
    就达不到分表的效果了。

<strong>根据以上结构，单实例但毫秒内请求低于512都不会出现重复的数字</strong>

<a id="idx-id-generator-system-arch-select"/>
## 4. ID生成器系统架构选择

4.1 使用Redis来做改造

> Redis是高性能的缓存服务器，这很符合ID生成器的要求，同时我们的业务中大量使用了Redis作为缓存，因此业务上比较容易接入。同时部署和运维上，Redis相对比较成熟，
> 其源代码也很优秀，改造成本低，减少出bug的可能

<a id="idx-exceed-shard-range-use-borrow-ms" />
## 5. 毫秒内超过shard-range请求次数时，采用借毫秒的方式超前分配ID

对于高并发，出现毫秒内超过shard-range的请求次数时，比如 shard-range = 9, 则 520 qpms, 520000 qps请求，这时，可以采用借未来的毫秒进行分配ID，这样即使超过了
分配的ID也是不重复的。

注意: 
    
> 若出现借秒的情况时，可以布置多个实例来平均分配压力，极端情况下，如果一直借毫秒，当server重启时，可能会出现ID重复的可能

<a id="idx-borrow-ms-algorithm" />
## 6. 借毫秒分配ID算法

类似于加法进位的原理，只是这里的进制是shard-range的大小，(以下假设shard-range = 9, shard-max = 2 ^ 9 = 512), 进制为512，当超过时，毫秒进位1：
```    
//当前时间
current_time = (mstime() - ID_START_TIMESTAMP);

//相对于当前时间，已经过去的shard数
shard_passed_relate_current = 0;
//向未来借的shard数
shard_future_relate_current = 0;

//当时间变更时，做一次进位变换
if(current_time != server.id_last_time){
    //已经过去的shard数为，(当前时间 - 上一次记录时间) * shard_max
    shard_passed_relate_current = (current_time - server.id_last_time) << server.shard_range;
    //未来借的shard数
    shard_future_relate_current = server.cur_shard - shard_passed_relate_current;
    //若借的shard数为负值，说明这段时间内，有部分shard未使用完成，因此清零
    server.cur_shard = shard_future_relate_current > 0 ? shard_future_relate_current : 0;
    //变换时间
    server.id_last_time = current_time;
}

//id真正的时间，当前时间 + 进位参数的时间(shard超过shard_max之后才有借位)
real_used_time = current_time + floor(server.cur_shard / server.shard_max);
//shard每次加一
server.cur_shard ++;
```

<a id="idx-id-generator-key-code" />
## 7. 关键代码实现
```c
// 文件: src/redis.c
static long long generateId() {
    long long current_time = (mstime() - ID_START_TIMESTAMP);
    long long shard_passed_relate_current = 0, shard_future_relate_current = 0;

    if(current_time != server.id_last_time){
        shard_passed_relate_current = (current_time - server.id_last_time) << server.shard_range;
        shard_future_relate_current = server.cur_shard - shard_passed_relate_current;
        server.cur_shard = shard_future_relate_current > 0 ? shard_future_relate_current : 0;
        server.id_last_time = current_time;
    }

    /**
     * generate id
     * should plus has borrowed ms from future
     */
    long long real_used_time = current_time + floor(server.cur_shard / server.shard_max);
    long long id = real_used_time << 21;
    id |= server.server_id << (21 - server.id_range);
    id |= server.cur_shard << (21 - server.id_range - server.shard_range);
    id |= server.random_sequences[server.cur_random];

    //rotate server.cur_random
    server.cur_random = (server.cur_random + 1) % server.random_sequence_max;
    server.cur_shard ++;
    return id;
}
```

<a id="idx-id-generator-redis-command" />
## 8. Redis命令支持
	
8.1. GETID 

	get one squenced id 

Return value

	the value of 64 bit squenced id

Examples:

```
redis> GETID
(integer) 144182182078841378
```


8.2. MGETID count

	get multi sequenced ids, the max count is 100

Return value

	list of sequenced ids.

Examples:

```
redis> MGETID 10
 1) "144182328657183537"
 2) "144182328657184499"
 3) "144182328657185133"
 4) "144182328657186040"
 5) "144182328657187118"
 6) "144182328657188278"
 7) "144182328657189411"
 8) "144182328657190557"
 9) "144182328657191123"
10) "144182328657192148"
```

<a id="idx-id-generator-press-test" />
## 9. 压测
机器配置:

    内存64G, 32核处理器, centos 7

测试程序:

```go
package main

import (
	"flag"
	"fmt"
	"github.com/garyburd/redigo/redis"
	"runtime"
	"sync"
	"time"
)

func Connect(address string, times int, wg *sync.WaitGroup) {
	defer wg.Done()
	wg.Add(1)
	conn, err := redis.Dial("tcp", address)
	if err != nil {
		return
	}
	//for i := 0; i < times; i++ {
	for {
		res, err := redis.Int(conn.Do("GETID"))
		//res, err := redis.Strings(conn.Do("MGETID", 100))
		
		if err != nil {
			fmt.Println(err.Error())
			continue
		}

		time.Sleep(time.Millisecond * 10)

		t := time.Now()
		log := fmt.Sprintf("%d-%d-%d %d:%d:%d %d", t.Year(), t.Month(), t.Day(), t.Hour(), t.Minute(), t.Second(), res)
		fmt.Println(log)
	}
}

func main() {
	runtime.GOMAXPROCS(-1)
	var num, times int
	flag.IntVar(&num, "num", 5, "ddddd")
	flag.IntVar(&times, "times", 1000, "times")
	flag.Parse()
	var wg sync.WaitGroup
	for i := 0; i < num; i++ {
		go Connect("host:6551", times, &wg)
	}
	time.Sleep(time.Second * 1)
	wg.Wait()
}

```

单ID请求: 最大qps: 9w qps左右, 不会产生借毫秒

多ID请求: 最大qps: 5000 qps左右，会产生借毫秒的情况

<strong>注: 若qps更高，应该多实例部署，平均压力到每个实例上，同时得接受这样产生的数据会在毫秒内不连续</strong>

<a id="idx-id-generator-service-deploy" />
## 10. ID生成器服务部署

```
git clone git@github.com:nice-opentech/id-generator-base-on-redis.git

编辑配置redis.conf:
	#id generator config
	# id total length is 63 bit, millisecond use 42 bit, so the remain bit is 21 bit
	# you can config server-id, id-range, id-shard, random-range freedom and confirm the sum of their bits is 21 bit
	#   id-range + shard-range + random-range = 21
	# server-id      0 ~ 3
	# id-range      use 2 bit  ;    0 ~ 3       ; 2^2 = 4
	# shard-range   use 9 bit  ;    0 ~ 511     ; 2^9 = 512
	# random-range  use 10 bit ;    0 ~ 1023    ; 2^10 = 1024
	
	id-generator server-id 0 id-range 2 shard-range 9 random-range 10

编译:
	make

启动服务:
	src/id-redis-server redis.conf
```

<a id="idx-id-generator-issues" />
## 9. issues

9.1 借毫秒警告

对于ID生成器的算法的局限:

	42bit(millisecond) + server_id_range(2bit, 可配置) + shard_range(9bit，可配置) + random_range(10bit, 可配置) = 63 bit;

单位毫秒内可使用的ID是有限的，若超出后，会有日志输出, 类似如下，向前借了1ms:

```
    [35695] 10 Feb 17:21:18.464 # warning > id=139842471327704136, ctime=66676878464, rtime=66681044992, bms=4166528, shard=2133262345, rand=72, sprc=0, sfrc=0, borrow=1
```

若使用过程中，经常出现该警告，说明qps > (2 ^ shard_range) * 1000; (如shard_range = 9，则会超过: 512000 qps才会出现借毫秒), 这时可以将random_range调小点，则可满足使用场景

