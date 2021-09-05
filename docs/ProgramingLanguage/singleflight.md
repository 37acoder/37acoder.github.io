# singleflight

git ref: https://github.com/golang/groupcache/blob/master/singleflight/singleflight.go

``` go
package singleflight

import "sync"

// call is an in-flight or completed Do call
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}

// Group represents a class of work and forms a namespace in which
// units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

singleflight是一个轻量级的防治缓存击穿的工具。

## 设计思路

### 数据结构

使用mutex和waitgroup来做并发控制和同步

`Group`结构是singleflight的主要api

``` go
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}
```

`mu`是互斥锁，用来保护 `m`这个map，防止并发读写map

`m`是一个map，其值是`call`。`m`里保存了当前正在执行的任务，可以通过m访问到某个正在进行的任务。

`call`结构是任务的类型

``` go
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}
```

`wg`是用来做任务同步的，正在执行任务的`goroutine`会给`wg`加1，其他`goroutine`会等待该`goroutine`执行完毕。

`val`,`err`是任务的输出，用来存储任务的结果。

### 主要逻辑

``` go
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

1. 执行加锁操作，锁住整个`m`
2. 从`m`获取到`key`对应的`call`
   1. 如果没有获取到，说明当前调用者是第一个执行该任务的。
      1. 新建一个`call`，并设置到`m`里面。这一步是为了让任务对其他调用者可见。
      2. `wg`增加1。这一步是为了让其他调用者感知任务正在执行中。
      3. 释放锁，交出`m`控制权。跳出临界区。
      4. 执行任务，并将结果设置到`call`里
      5. `wg`减1。让其他调用者感知任务已执行完毕。其他调用者结束阻塞。
      6. 再次锁住`m`，删除掉掉这个`key`，并释放锁。
      7. 返回结果
   2. 如果获取到，说明当前调用者不是第一个执行该任务的，有其他任务已经在执行该任务。
      1. 释放锁，等待`wg`。待第一个执行任务者执行结束。
      2. 直接返回任务的执行结果。

## 思考

1. 第一个调用者为什么要从map里删除这个key?

   singleflight解决的问题是，避免**同一时间**有多个调用者执行重复的操作，而不解决**缓存问题**。删除掉该key是为了让之后的调用者能够重新执行任务，而不是一直使用之前的任务结果。如果一直使用之前的任务结果，那singleflight就是一个单例实现的缓存；实际的场景任务是用来更新缓存的，而不是实现缓存

2. 怎么保证删除掉key后，其他的调用者还能获取到任务结果？

   任务执行中的时候，任务对应的`key`在`m`里能够访问到，其他调用者访问到并持有的是 `call`。只删除key不会导致value的改变，其他调用者依然能获取到当时的任务结果。

