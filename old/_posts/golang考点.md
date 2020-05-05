---
title: go 考点list
date: 2019-06-13 21:55:41
tags: go,内存
---

1 list清单
===
调度模型MPG：若G无限多，调度困难，gc回收压力大，性能下降，所以推荐使用go poll
channel底层实现：基于锁，发送接收队列，堵塞goroutine(设置goroutine状态为堵塞)；memcopy，内存实现；
go routine底层调度:MPG,通过go调用函数时抢占式调用，若go堵塞（chan或者netpoll)则从M分离，入到相应的堵塞队列；
go内存模型：栈模型，chan, 锁
gc垃圾回收
逃逸分析：变量被其他函数引用
sync atom包
runtime包


go mod
