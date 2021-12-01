---
layout: post
title:  "Golang race demo"
date:   2021-12-01 
categories: golang
---
```

import (
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

// 有data race
func TestDemo1(t *testing.T) {
	running := true
	go func() {
		t.Log("thread1 start")
		count := 1
		for running {
			count++
		}
		t.Log("thread1 end : count = ", count) //这个循环永远也不会结束,为什么？
	}()
	go func() {
		t.Log("start thread2")
		for {
			running = false
		}
	}()
	time.Sleep(10 * time.Second)
	return
}

// 无data race
func TestSafeDemo1(t *testing.T) {
	var running *int32 = new(int32)
	go func() {
		t.Log("thread1 start")
		count := 1
		for atomic.LoadInt32(running) == 0 {
			count++
		}
		t.Log("thread1 end : count = ", count) //这个循环会结束
	}()
	go func() {
		t.Log("start thread2")
		for {
			atomic.StoreInt32(running, 1)
		}
	}()
	time.Sleep(10 * time.Second)
	return
}

type SingletonInt32 struct {
	i   int32
	mtx sync.Mutex
}

func (a *SingletonInt32) Instance() int32 {
	if a.i == 0 { // 使用 go race detect 可以检测到datarace，和60行
		a.mtx.Lock()
		defer a.mtx.Unlock()
		if a.i == 0 {
			a.i = 1 // 使用 go race detect 可以检测到datarace，和56行
		}
	}
	return a.i
}

// 有data race
func TestDemo2(t *testing.T) {
	s := &SingletonInt32{}
	for i := 0; i < 100; i++ {
		go func(ii int) {
			t.Logf("seq %d: %d", ii, s.Instance())
		}(i)
	}
	time.Sleep(10 * time.Second)
}

type SafeSingletonInt32 struct {
	i   int32
	mtx sync.Mutex
}

func (a *SafeSingletonInt32) Instance() int32 {
	if atomic.LoadInt32(&a.i) == 0 {
		a.mtx.Lock()
		defer a.mtx.Unlock()
		if atomic.LoadInt32(&a.i) == 0 {
			atomic.StoreInt32(&a.i, 1)
		}
	}
	return a.i
}

// 无data race
func TestSafeDemo2(t *testing.T) {
	s := &SafeSingletonInt32{}
	for i := 0; i < 100; i++ {
		go func(ii int) {
			t.Logf("seq %d: %d", ii, s.Instance())
		}(i)
	}
	time.Sleep(10 * time.Second)
}

```
golang不同协程操作同一个变量要注意可见性问题，必要时可使用go的race detect，很好用。

### Reference
1. [Golang Memory Model](https://www.codenong.com/js1596e1d7c126/)
2. [Golang Memory Model](https://go.dev/ref/mem)
