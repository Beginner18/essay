---
title: go-list
date: 2019-09-12 11:05:44
tags:
---
Go踩过的坑
====

1. 依赖问题
===
1.1 golang.org/x/net/trace依赖重复  
---
[stackOverflow] (https://github.com/etcd-io/etcd/issues/9357)

>- 问题描述：
>
```
panic: /debug/requests is already registered. You may have two independent copies of golang.org/x/net/trace in your binary, trying to maintain separate state. This may involve a vendored copy of golang.org/x/net/trace.
```
>- 策略：
>
>>- find /go/src/ -patch "*net/trace"找到所有的net/trace包
>>- rm -rf xx/net/trace; 保留go/src包中一个
>>
```
/go/src/go.etcd.io/etcd/vendor/golang.org/x/net/trace
/go/src/go.etcd.io/etcd/vendor/golang.org/x/net/trace/events.go
/go/src/go.etcd.io/etcd/vendor/golang.org/x/net/trace/histogram.go
/go/src/go.etcd.io/etcd/vendor/golang.org/x/net/trace/trace.go
/go/src/github.com/coreos/etcd/cmd/vendor/golang.org/x/net/trace
/go/src/github.com/coreos/etcd/cmd/vendor/golang.org/x/net/trace/events.go
/go/src/github.com/coreos/etcd/cmd/vendor/golang.org/x/net/trace/histogram.go
/go/src/github.com/coreos/etcd/cmd/vendor/golang.org/x/net/trace/trace.go
/go/src/golang.org/x/net/trace
/go/src/golang.org/x/net/trace/events.go
/go/src/golang.org/x/net/trace/histogram.go
/go/src/golang.org/x/net/trace/trace_test.go
/go/src/golang.org/x/net/trace/trace.go
/go/src/golang.org/x/net/trace/histogram_test.go
```

>- 问题剖析：net/trace包init函数包括相关handler的注册，若重复引用则重复注册handler引起panic问题
>
>>
```
net/trace包：
func init() {
// 注册default serverMux：handler已经存在重复注册panic
	http.HandleFunc("/debug/requests", func(w http.ResponseWriter, req *http.Request) {
		any, sensitive := AuthRequest(req)
		if !any {
			http.Error(w, "not allowed", http.StatusUnauthorized)
			return
		}
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		Render(w, req, sensitive)
	})
	http.HandleFunc("/debug/events", func(w http.ResponseWriter, req *http.Request) {
		any, sensitive := AuthRequest(req)
		if !any {
			http.Error(w, "not allowed", http.StatusUnauthorized)
			return
		}
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		RenderEvents(w, req, sensitive)
	})
}
```