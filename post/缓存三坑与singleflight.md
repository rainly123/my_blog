---
title: "缓存三坑与singleflight"
date: 2022-01-14T15:39:00+08:00
description: java，golang工程师，项目管理，软件架构
toc: true
draft: true
---

## 缓存的坑
- **缓存雪崩**：缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的 key 设置了相同的过期时间等引起。

- **缓存击穿**：一个存在的key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到 DB ，造成瞬时DB请求量大、压力骤增。
- **缓存穿透**：查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求 DB，如果瞬间流量过大，穿透到 DB，导致宕机。

## 如何避免
![cache-miss](../images/cache-miss.png)
我们大多数时候的缓存策略是先从redis里面去读取数据，读取不到的情况下再去下游存储服务，因此高并发场景下，一旦出现雪崩，击穿，穿透这三种问题，存储服务将面临非常严峻的考验，因此严格限制对下游存储服务的并发数量也变得非常关键。
golang官方给我们提供了sync.singlefilght这个包，由此，缓存策略可以优化成这样：
![singleflight](../images/singleflight.png)
在绝大多数场景下，singleflight 都很好用，因此让很多人相信 singleflight 是完美无缺的银弹。在2020年的电商大促中，有朋友因为此种认知，导致线上业务出现了严重故障。

## singleflight科学实用
 - singleflight标准库：golang.org/x/sync/singleflight
 - singleflight提供了以下三个方法:
```golang
// Do(): 相同的 key，fn 同时只会执行一次，返回执行的结果给fn执行期间，所有使用该 key 的调用
// v: fn 返回的数据
// err: fn 返回的err
// shared: 表示返回数据是调用 fn 得到的还是其他相同 key 调用返回的
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
// DoChan(): 类似Do方（），以 chan 返回结果
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
// Forget(): 失效 key，后续对此 key 的调用将执行 fn，而不是等待前面的调用完成
func (g *Group) Forget(key string)
```

### 例子
- 通常的使用方法如下
```golang

package main

import (
	"context"
	"fmt"
	"golang.org/x/sync/singleflight"
	"sync/atomic"
	"time"
)

type Result string

func find(ctx context.Context, query string) (Result, error) {
	return Result(fmt.Sprintf("result for %q", query)), nil
}

func main() {
	var g singleflight.Group
	const n = 5
	waited := int32(n)
	done := make(chan struct{})
	key := "https://weibo.com/1227368500/H3GIgngon"
	for i := 0; i < n; i++ {
		go func(j int) {
			v, _, shared := g.Do(key, func() (interface{}, error) {
				ret, err := find(context.Background(), key)
				return ret, err
			})
			if atomic.AddInt32(&waited, -1) == 0 {
				close(done)
			}
			fmt.Printf("index: %d, val: %v, shared: %v\n", j, v, shared)
		}(i)
	}

	select {
	case <-done:
	case <-time.After(time.Second):
		fmt.Println("Do hangs")
	}
}
```

- 输出结果

```g
index: 1, val: result for "https://weibo.com/1227368500/H3GIgngon", shared: true
index: 2, val: result for "https://weibo.com/1227368500/H3GIgngon", shared: true
index: 3, val: result for "https://weibo.com/1227368500/H3GIgngon", shared: false
index: 4, val: result for "https://weibo.com/1227368500/H3GIgngon", shared: false
index: 0, val: result for "https://weibo.com/1227368500/H3GIgngon", shared: false
```
如果一切正常，所有的请求都能够获取正确的数据，但是如果函数执行遇到问题呢？由于singleflight是以阻塞的方式读取下游请求的并发量，在第一个下游请求没返回之前，所有的请求都会被阻塞。

## 分析

假设服务正常情况下处理能力为 1W QPS，每次请求会发起 3 次 下游调用，其中一个下游调用使用 `singleflight` 获取控制并发获取数据，请求超时时间为3S。那么在出现请求超时的情况下，会出现以下几个问题：

- 协程暴增，最小协程数为3W（1 W/S * 3S）

- 内存暴涨，内存总大小为：协程内存大小 + 1W/S * 3S *（3+1）次 * （请求包+响应包）大小

- 大量超时报错：1W/S * 3S

- 后续请求耗时增加（调度等待）

出现上述问题的根本原因是以下两点：
1. 阻塞读：缺少超时控制，难以快速失败；
2. 单并发：虽然达到了控制并发的效果，但是牺牲了成功率；

## 如何应对
1. **超时控制** ，用DoChan替代Do，DoChan() 通过 channel 返回结果。因此可以使用 select 语句实现超时控制
```golang
ch := g.DoChan(key, func() (interface{}, error) {
    ret, err := find(context.Background(), key)
    return ret, err
})
// Create our timeout
timeout := time.After(500 * time.Millisecond)

var ret singleflight.Result
select {
case <-timeout: // Timeout elapsed
        fmt.Println("Timeout")
    return
case ret = <-ch: // Received result from channel
    fmt.Printf("index: %d, val: %v, shared: %v\n", j, ret.Val, ret.Shared)
}
```
2. **降低请求数量** 
在一些对可用性要求极高的场景下，往往需要一定的请求饱和度来保证业务的最终成功率。一次请求还是多次请求，对于下游服务而言并没有太大区别，此时使用 `singleflight` 只是为了降低请求的数量级，那么使用 Forget() 提高下游请求的并发:

```
   v, _, shared := g.Do(key, func() (interface{}, error) {
       go func() {
           time.Sleep(10 * time.Millisecond)
           fmt.Printf("Deleting key: %v\n", key)
           g.Forget(key)
       }()
       ret, err := find(context.Background(), key)
       return ret, err
   })
```

当有一个并发请求超过 10ms，那么将会有第二个请求发起，此时只有 10ms 内的请求最多发起一次请求，即最大并发：100 QPS。单次请求失败的影响大大降低。







