---
layout: post
title:  "Golang race demo"
date:   2021-12-01 
categories: golang
---
```
package test

import (
	"sync/atomic"
	"testing"
	"time"
)

func TestRace(t *testing.T) {
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
	time.Sleep(time.Hour)
	return
}

func TestRaceSafe(t *testing.T) {
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
	time.Sleep(time.Hour)
	return
}

```
golang不同协程操作同一个变量要注意可见性问题

### Reference
1. [Golang Memory Model](https://www.codenong.com/js1596e1d7c126/)
2. [Golang Memory Model](https://go.dev/ref/mem)
