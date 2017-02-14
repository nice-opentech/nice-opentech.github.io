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

### 目录

1. [63位整形ID生成器算法](#idx-int-63-bit-id-generate-algorithm)

2. [ID生成器提供的命令](#idx-support-command)

3. [为何选择redis做改造](#idx-why-choose-redis)

4. [极端问题考虑](#idx-extreme-emergency-think)

5. [生成ID数据统计](#idx-statistics)

<a id="idx-int-63-bit-id-generate-algorithm" />

<hr/>
### 1.63位整形ID生成器算法

```c
// 文件: src/redis.c
static long long generateId() {
    long long current_time = (mstime() - ID_START_TIMESTAMP);
    long long shard_passed_relate_current = 0, shard_future_relate_current = 0;

    if(current_time != server.id_last_time){
        shard_passed_relate_current = (current_time - server.id_last_time) << server.shard_range;

        /**
         * if (shard_future_relate_current > 0) {
         *      we have borrowed some ms from future
         * } eles if (shard_future_relate_current < 0){
         *      duration from last time to current time, no request
         * } else if (shard_future_relate_current == 0){
         *      last time shard range has run out, so the new ms can service and not borrow ms from future
         * }
         * so server.cur_shard = maxInt(0, shard_future_relate_current) mean:
         *  if(shard_future_relate_current > 0) {
         *      we have borrowed some ms, so the real_used_time will plus borrowed ms ->
         *          real_used_time = current_time + floor(server.cur_shard / server.shard_max);
         *  } else if(shard_future_relate_current <=0){
         *      some has passed ms not used, so ignore it and record from current
         *  }
         */
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

这个函数是生成ID的整个流程，去掉了写日志的逻辑，总体算法如下:

1.	id构成

	总长度63位，因为php的是有符号整数

	42bit(millisecond) + server_id_range(2bit, 可配置) + shard_range(9bit，可配置) + random_range(10bit, 可配置) = 63 bit;

	说明: 

    1. 配置必须满足: server_id_range + shard_range + random_range = 21 bit
    2. 前42bit: 是毫秒的时间戳 
    3. server_id_range: 用于布置多台实例，如果只布置一个，可以用一位，那shard_range就可以多一位了,
    4. shard_range: 单个毫秒内可用的不重复的ID总数, 9bit = 2^9 = 512，即一毫秒最多产生512个ID
    5. random_range: 随机数，因为目前我们的表是1024个分表，所以采用了2^10 = 1024个数，这样读取的数据最终可以均匀的分配到以ID取模的所有分表中

2.	函数算法

	总体分为四步:

    2.1. 获取当前的毫秒数，并初始化向未来借秒数 = 0
    
    2.2. 当毫秒变化时，对shard的处理，这里的逻辑主要是可能向未来借毫秒的作用
```
    1. shard_passed_relate_current = (current_time - server.id_last_time) << server.shard_range		
        获取上一毫秒到当前毫秒已经过了的shard总数，一个毫秒就是用幂函数表示是: 2^server.shard_range，
        为了运算更快，使用移位 << server.shard_range
    
    2. shard_future_relate_current = server.cur_shard - shard_passed_relate_current;

        若shard_future_relate_current > 0:
        	表明向未来借了毫秒了，因为上一毫秒到当前毫秒的ID已经被使用完了，因此产生
            新的ID时，需要从新的shard开始计算，即要算未来的时间
    
        若shard_future_relate_current <= 0:
        	表明上一毫秒的ID还未使用完
    
    3. server.cur_shard = shard_future_relate_current > 0 ? shard_future_relate_current : 0;
        变换cur_shard值，server.cur_shard开始时初始化为0, 每次生成新的ID后会++[见倒数第二行]，
        该值代表了获取到的ID数，因为时间在更新，所以可以将cur_shard和时间关联上，每次不同毫秒
        均会对cur_shard进行处理，变更情况如下:
    
        若未产生借毫秒 shard_future_relate_current <=0, 则cur_shard = 0，表示该毫秒内的ID未被使用过
    
        若产生借毫秒 shard_future_relate_current >0, 则cur_shard = shard_future_relate_current，
            表示该毫秒内的ID被使用过, 或未来的毫秒被使用了
```
    2.3. 生成ID
    
    	long long real_used_time = current_time + floor(server.cur_shard / server.shard_max);

    	由于有可能产生借毫秒的可能，单位毫秒时间超过shard_range的总数, (eg: shard_range = 9, 
        则一毫秒超过512个ID时会产生借毫秒), 因此应该加上借的毫秒数

    	server.shard_max = 1 << shard_range; 即毫秒内的ID总数，这里的即毫秒的意义类似于加法进位
        的概念，只是这里的进位是shard_range的总数，(eg: shard_range = 9, 这进制是512)

    	long long id = real_used_time << 21;
    	将毫秒右移21位，这样后面的21位供余下的数据填充

    	id |= server.server_id << (21 - server.id_range); 
    	或上server_id，只右移(21 - server.id_range)位

    	id |= server.cur_shard << (21 - server.id_range - server.shard_range);
    	或上server.cur_shard，只右移(21 - server.id_range - server.shard_range)位

    	id |= server.random_sequences[server.cur_random];
    	或上随机数，该随机数时程序启动时，初始化的随机打乱的总长度为2^random_range个元素的数组，
        后续直接通过顺序访问随机数组即可达到随机的状态			
    
    
    2.4. 随机数做rotate，不停的循环使用, cur_shard++，表示生成了一个ID


<a id="idx-support-command" />

### 2.ID生成器提供的命令

1. 单个ID提取
	getid
2. 多个ID提取, 一次最多提取100个，超过则返回错误
	mgetid	count

<a id="idx-why-choose-redis" />


### 3.为何选择redis做改造

1. 生成ID不需要多线程模型，单线程即可，redis比较符合计算，因为单线程处理少了很多多线程的坑

2. redis的协议已经封装好了的，并且现在所有的业务都使用该通用组件，只需要在php的redis扩展或
      其他语言的扩展中加两个命令即可以很好的融合到业务代码中

3. redis足够高效，因为本身ID生成器可能需要抗大量的并发，但这正是redis的特长


<a id="idx-extreme-emergency-think" />

### 4.极端问题考虑

对于ID生成器的算法的局限:

	总长度63位，因为php的是有符号整数

	42bit(millisecond) + server_id_range(2bit, 可配置) + shard_range(9bit，可配置) + random_range(10bit, 可配置) = 63 bit;

单位毫秒内可使用的ID是有限的，若超出后，会有日志输出, 类似如下，向前借了1ms:
```
[35695] 10 Feb 17:21:18.464 # warning > id=139842471327704136, ctime=66676878464, rtime=66681044992, bms=4166528, shard=2133262345, rand=72, sprc=0, sfrc=0, borrow=1
```
若使用过程中，经常出现该警告，说明qps > (2 ^ shard_range) * 1000; (如shard_range = 9，则会超过: 512000 qps才会出现借毫秒), 这时可以将random_range调小点，则可满足使用场景


<a id="idx-statistics" />

### 5.生成ID数据统计
机器配置:

    内存64G, 32核处理器, centos 7

使用go语言做高并发请求，发送redis命令:

单ID请求: 最大qps: 9w qps左右, 不会产生借毫秒
```
redis-cli -h host -p port getid
```

多ID请求: 最大qps: 5000 qps左右，会产生借毫秒的情况
```
redis-cli -h host -p port getid 100
```
