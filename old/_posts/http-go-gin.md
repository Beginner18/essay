---
title: http-go-gin
date: 2019-07-17 17:01:59
categories: "网络框架" #分类
archives: "近期"
tags:
	- http
	- gin
	
---

1 [简介](https://github.com/gin-gonic/gin)
====
>- httpRoute架构非MVC(beego)
>- handler为radix tree结构，GET POST...不同方法为不同树
>- 压缩字母树：自动分裂为新子树
>- 中间件处理：中间件为[]handler中的前几个handler;最后一个handler为响应路径对应的处理方式，中间件函数内添加c.Next()执行下一个hadler函数；链式处理
>- 底层：listenAndServer等均为go底层net/http
[gin参考文档](https://juejin.im/post/5c7c923cf265da2dce1f5fe8)
2 源码分析
====
***Engine***
>- 负责路由、中间件等

```
// Engine is the framework's instance, it contains the muxer, middleware and configuration settings.
// Create an instance of Engine, by using New() or Default()
type Engine struct {
	RouterGroup

	// Enables automatic redirection if the current route can't be matched but a
	// handler for the path with (without) the trailing slash exists.
	// For example if /foo/ is requested but a route only exists for /foo, the
	// client is redirected to /foo with http status code 301 for GET requests
	// and 307 for all other request methods.
	RedirectTrailingSlash bool

	// If enabled, the router tries to fix the current request path, if no
	// handle is registered for it.
	// First superfluous path elements like ../ or // are removed.
	// Afterwards the router does a case-insensitive lookup of the cleaned path.
	// If a handle can be found for this route, the router makes a redirection
	// to the corrected path with status code 301 for GET requests and 307 for
	// all other request methods.
	// For example /FOO and /..//Foo could be redirected to /foo.
	// RedirectTrailingSlash is independent of this option.
	RedirectFixedPath bool

	// If enabled, the router checks if another method is allowed for the
	// current route, if the current request can not be routed.
	// If this is the case, the request is answered with 'Method Not Allowed'
	// and HTTP status code 405.
	// If no other Method is allowed, the request is delegated to the NotFound
	// handler.
	HandleMethodNotAllowed bool
	ForwardedByClientIP    bool

	// #726 #755 If enabled, it will thrust some headers starting with
	// 'X-AppEngine...' for better integration with that PaaS.
	AppEngine bool

	// If enabled, the url.RawPath will be used to find parameters.
	UseRawPath bool

	// If true, the path value will be unescaped.
	// If UseRawPath is false (by default), the UnescapePathValues effectively is true,
	// as url.Path gonna be used, which is already unescaped.
	UnescapePathValues bool

	// Value of 'maxMemory' param that is given to http.Request's ParseMultipartForm
	// method call.
	MaxMultipartMemory int64

	delims           render.Delims
	secureJsonPrefix string
	HTMLRender       render.HTMLRender
	FuncMap          template.FuncMap
	allNoRoute       HandlersChain
	allNoMethod      HandlersChain
	noRoute          HandlersChain
	noMethod         HandlersChain
	// 线程安全，维护空闲列表，缓存已分配内存但暂时不用的项方便后续再利用，减少gc及内存分	// 配压力,适合管理一个package中并发对立的client(http request)共享&再利用的临时	// 项，	pool可以缓冲分配开销；不适合short-lived object
	// 此处用于gin.Context缓冲池
	pool             sync.Pool
	trees            methodTrees
}
type methodTree struct {
	method string
	root   *node
}
type methodTrees []methodTree
type node struct {
	//path为绝对路径
	path      string
	indices   string
	children  []*node
	// handlers为对应path的所有handlers
	handlers  HandlersChain
	priority  uint32
	nType     nodeType
	maxParams uint8
	wildChild bool
}
// New returns a new blank Engine instance without any middleware attached.
// By default the configuration is:
// - RedirectTrailingSlash:  true
// - RedirectFixedPath:      false
// - HandleMethodNotAllowed: false
// - ForwardedByClientIP:    true
// - UseRawPath:             false
// - UnescapePathValues:     true
func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
		ForwardedByClientIP:    true,
		AppEngine:              defaultAppEngine,
		UseRawPath:             false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
		trees:                  make(methodTrees, 0, 9),
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJsonPrefix:       "while(1);",
	}
	engine.RouterGroup.engine = engine
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
	return engine
}
```
[golang pool简介](https://blog.csdn.net/yongjian_lian/article/details/42058893)
>
> ***serverHTTP***(net/http包handler的实现)
>
```
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	// c.index初始化为-1
	c.reset()
	engine.handleHTTPRequest(c)
	engine.pool.Put(c)
}
func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method
	path := c.Request.URL.Path
	unescape := false
	if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
		path = c.Request.URL.RawPath
		unescape = engine.UnescapePathValues
	}
	// Find root of the tree for the given HTTP method
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method == httpMethod {
			root := t[i].root
			// Find route in tree
			handlers, params, tsr := root.getValue(path, c.Params, unescape)
			if handlers != nil {
				c.handlers = handlers
				c.Params = params
				// 执行handlerChain，即中间件及实际处理
				c.Next()
				// 写回conn返回http结果
				c.writermem.WriteHeaderNow()
				return
			}
			if httpMethod != "CONNECT" && path != "/" {
				if tsr && engine.RedirectTrailingSlash {
					redirectTrailingSlash(c)
					return
				}
				if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
					return
				}
			}
			break
		}
	}
	if engine.HandleMethodNotAllowed {
		for _, tree := range engine.trees {
			if tree.method != httpMethod {
				if handlers, _, _ := tree.root.getValue(path, nil, unescape); handlers != nil {
					c.handlers = engine.allNoMethod
					serveError(c, 405, default405Body)
					return
				}
			}
		}
	}
	c.handlers = engine.allNoRoute
	serveError(c, 404, default404Body)
}
```
>
***RouteGroup***  

>- gin可以设置路由组，所有路由组下路由绝对路径为RouteGroup.basePath+relativePath;
>- 中间件：路由组下所有路由的中间件为路由本身中间件+路由组中间件
>- Handlers即为路由组相应中间件，最后一个可为具体请求处理
>- 新建routeGroup中的engine为原engine的地址一致
>- engine中的trees为完整的路由树，routeGrouop方便按组添加中间件及某一路径下多子路径添加
>
```
// RouterGroup is used internally to configure router, a RouterGroup is associated with a prefix
// and an array of handlers (middleware).
type RouterGroup struct {
	Handlers HandlersChain
	basePath string
	engine   *Engine
	root     bool
}
// handlerFun定义
type HandlerFunc func(*Context)
// 组下添加handler
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}
// Use adds middleware to the group, see example code in github.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```
>- 每个路径还可以单独添加自己独有的中间件,通过(RouteGroup.Handle):  
>
```
// handlers可以添加某个relativePath独有的中间件，最终该path的中间件为其所在组的group.handlers+输入参数handlers
func (group *RouterGroup) Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes {
	if matches, err := regexp.MatchString("^[A-Z]+$", httpMethod); !matches || err != nil {
		panic("http method " + httpMethod + " is not valid")
	}
	return group.handle(httpMethod, relativePath, handlers)
}
```
>- 创建新的routeGroup:
>
```
// Group creates a new router group. You should add all the routes that have common middlwares or the same path prefix.
// For example, all the routes that use a common middlware for authorization could be grouped.
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
}
```
***GIN Context***
>- listen建立accept后，针对每个请求执行handler时新建gin.Context
>- httpReq项:通过query postfrom bind等方法解析
>
```
// Param returns the value of the URL param.
// It is a shortcut for c.Params.ByName(key)
//     router.GET("/user/:id", func(c *gin.Context) {
//         // a GET request to /user/john
//         id := c.Param("id") // id == "john"
//     })
func (c *Context) Param(key string) string {
	return c.Params.ByName(key)
}
// Bind checks the Content-Type to select a binding engine automatically,
// Depending the "Content-Type" header different bindings are used:
//     "application/json" --> JSON binding
//     "application/xml"  --> XML binding
// otherwise --> returns an error.
// It parses the request's body as JSON if Content-Type == "application/json" using JSON or XML as a JSON input.
// It decodes the json payload into the struct specified as a pointer.
// It writes a 400 error and sets Content-Type header "text/plain" in the response if input is not valid.
func (c *Context) Bind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.MustBindWith(obj, b)
}
// Default returns the appropriate Binding instance based on the HTTP method
// and the content type.
// 根据context-type选择不同的解析engine(不同接口实现)
func Default(method, contentType string) Binding {
	if method == "GET" {
		return Form
	}
	switch contentType {
	case MIMEJSON:
		return JSON
	case MIMEXML, MIMEXML2:
		return XML
	case MIMEPROTOBUF:
		return ProtoBuf
	case MIMEMSGPACK, MIMEMSGPACK2:
		return MsgPack
	default: //case MIMEPOSTForm, MIMEMultipartPOSTForm:
		return Form
	}
}
```
>- http1串行处理一个连接的http request
>- 通过net/http包的response结构体实现net/http包的ResponseWriter接口，原生net/http包读到request请求后建立response结构体
>- 	server函数处理request请求时调用serverhandler.ServerHTTP方法，其中responseWriter接口的实现为(c *conn)readRequest返回的response结构体
>
```
// Serve a new connection.
func (c *conn) serve(ctx context.Context) {
	...省略
		// HTTP cannot have multiple simultaneous active requests.[*]
		// Until the server replies to this request, it can't read another,
		// so we might as well run the handler in this goroutine.
		// [*] Not strictly true: HTTP pipelining. We could let them all 		// process in parallel even if their responses need to be serialized.
		// But we're not going to implement HTTP pipelining because it
		// was never deployed in the wild and the answer is HTTP/2.
		serverHandler{c.server}.ServeHTTP(w, w.req)
		...省略
}
```
>- 处理http请求，获取请求 返回结果
>
```
// Read next request from connection.
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
	if c.hijacked() {
		return nil, ErrHijacked
	}
	var (
		wholeReqDeadline time.Time // or zero if none
		hdrDeadline      time.Time // or zero if none
	)
	t0 := time.Now()
	if d := c.server.readHeaderTimeout(); d != 0 {
		hdrDeadline = t0.Add(d)
	}
	if d := c.server.ReadTimeout; d != 0 {
		wholeReqDeadline = t0.Add(d)
	}
	c.rwc.SetReadDeadline(hdrDeadline)
	if d := c.server.WriteTimeout; d != 0 {
		defer func() {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}()
	}
	c.r.setReadLimit(c.server.initialReadLimitSize())
	if c.lastMethod == "POST" {
		// RFC 7230 section 3 tolerance for old buggy clients.
		peek, _ := c.bufr.Peek(4) // ReadRequest will get err below
		c.bufr.Discard(numLeadingCRorLF(peek))
	}
	req, err := readRequest(c.bufr, keepHostHeader)
	if err != nil {
		if c.r.hitReadLimit() {
			return nil, errTooLarge
		}
		return nil, err
	}
	if !http1ServerSupportsRequest(req) {
		return nil, badRequestError("unsupported protocol version")
	}
	c.lastMethod = req.Method
	c.r.setInfiniteReadLimit()
	hosts, haveHost := req.Header["Host"]
	isH2Upgrade := req.isH2Upgrade()
	if req.ProtoAtLeast(1, 1) && (!haveHost || len(hosts) == 0) && !isH2Upgrade && req.Method != "CONNECT" {
		return nil, badRequestError("missing required Host header")
	}
	if len(hosts) > 1 {
		return nil, badRequestError("too many Host headers")
	}
	if len(hosts) == 1 && !httpguts.ValidHostHeader(hosts[0]) {
		return nil, badRequestError("malformed Host header")
	}
	for k, vv := range req.Header {
		if !httpguts.ValidHeaderFieldName(k) {
			return nil, badRequestError("invalid header name")
		}
		for _, v := range vv {
			if !httpguts.ValidHeaderFieldValue(v) {
				return nil, badRequestError("invalid header value")
			}
		}
	}
	delete(req.Header, "Host")
	ctx, cancelCtx := context.WithCancel(ctx)
	req.ctx = ctx
	req.RemoteAddr = c.remoteAddr
	req.TLS = c.tlsState
	if body, ok := req.Body.(*body); ok {
		body.doEarlyClose = true
	}
	// Adjust the read deadline if necessary.
	if !hdrDeadline.Equal(wholeReqDeadline) {
		c.rwc.SetReadDeadline(wholeReqDeadline)
	}
	// ***建立response***
	// 通过(c conn)server中serverHandler{c.server}.ServeHTTP(w, w.req)实现ResponseWriter接口
	// http1.1非并发处理一个连接的请求，http2.0实现，http2.0使用不够广泛
	w = &response{
		conn:          c,
		cancelCtx:     cancelCtx,
		req:           req,
		reqBody:       req.Body,
		handlerHeader: make(Header),
		contentLength: -1,
		closeNotifyCh: make(chan bool, 1),
		// We populate these ahead of time so we're not
		// reading from req.Header after their Handler starts
		// and maybe mutates it (Issue 14940)
		wants10KeepAlive: req.wantsHttp10KeepAlive(),
		wantsClose:       req.wantsClose(),
	}
	if isH2Upgrade {
		w.closeAfterReply = true
	}
	w.cw.res = w
	w.w = newBufioWriterSize(&w.cw, bufferBeforeChunkingSize)
	return w, nil
}
```
>
>- context存放httpReq获取的请求参数及responseWrite将返回结果返回请求方
>- 
>
```
// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter
	// 存储url参数
	Params   Params
	handlers HandlersChain
	index    int8
	engine *Engine
	// Keys is a key/value pair exclusively for the context of each request.
	// 可以通过set/get方法设置及获取keys
	Keys map[string]interface{}
	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs
	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string
}
```
>- context的初始化reset,从poll中取出后reset
>
```
func (c *Context) reset() {
	c.Writer = &c.writermem
	c.Params = c.Params[0:0]
	c.handlers = nil
	// index初始化为-1
	c.index = -1
	c.Keys = nil
	c.Errors = c.Errors[0:0]
	c.Accepted = nil
}
```
>- 不同goroutine引用context时需要copy context副本，保证并发安全
>
```
// Copy returns a copy of the current context that can be safely used outside the request's scope.
// This has to be used when the context has to be passed to a goroutine.
func (c *Context) Copy() *Context {
	var cp = *c
	cp.writermem.ResponseWriter = nil
	cp.Writer = &cp.writermem
	cp.index = abortIndex
	cp.handlers = nil
	return &cp
}
```
>- 获取主handler
>
```
// Handler returns the main handler.
func (c *Context) Handler() HandlerFunc {
	return c.handlers.Last()
}
```
>- context的next处理
>
```
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in GitHub.
func (c *Context) Next() {
	c.index++
	for s := int8(len(c.handlers)); c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}
```
>- recover中间件示例
>
```
// Recovery returns a middleware that recovers from any panics and writes a 500 if there was one.
func Recovery() HandlerFunc {
	return RecoveryWithWriter(DefaultErrorWriter)
}
// RecoveryWithWriter returns a middleware for a given writer that recovers from any panics and writes a 500 if there was one.
func RecoveryWithWriter(out io.Writer) HandlerFunc {
	var logger *log.Logger
	if out != nil {
		logger = log.New(out, "\n\n\x1b[31m", log.LstdFlags)
	}
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				if logger != nil {
					stack := stack(3)
					httprequest, _ := httputil.DumpRequest(c.Request, false)
					logger.Printf("[Recovery] %s panic recovered:\n%s\n%s\n%s%s", timeFormat(time.Now()), string(httprequest), err, stack, reset)
				}
				c.AbortWithStatus(500)
			}
		}()
		// 链式处理handler
		c.Next()
	}
}
// 中断handlers链式处理
// Abort prevents pending handlers from being called. Note that this will not stop the current handler.
// Let's say you have an authorization middleware that validates that the current request is authorized.
// If the authorization fails (ex: the password does not match), call Abort to ensure the remaining handlers
// for this request are not called.
func (c *Context) Abort() {
	c.index = abortIndex
}
```
>- http response不同返回类型render处理&&设置cookie
>
```
func (c *Context) Render(code int, r render.Render) {
	c.Status(code)
	if !bodyAllowedForStatus(code) {
		r.WriteContentType(c.Writer)
		c.Writer.WriteHeaderNow()
		return
	}
	if err := r.Render(c.Writer); err != nil {
		panic(err)
	}
}
// HTML renders the HTTP template specified by its file name.
// It also updates the HTTP code and sets the Content-Type as "text/html".
// See http://golang.org/doc/articles/wiki/
func (c *Context) HTML(code int, name string, obj interface{}) {
	// 获取render实现
	instance := c.engine.HTMLRender.Instance(name, obj)
	// render返回结果
	c.Render(code, instance)
}
// render接口
type Render interface {
	Render(http.ResponseWriter) error
	WriteContentType(w http.ResponseWriter)
}
// render枚举实现
var (
	_ Render     = JSON{}
	_ Render     = IndentedJSON{}
	_ Render     = SecureJSON{}
	_ Render     = JsonpJSON{}
	_ Render     = XML{}
	_ Render     = String{}
	_ Render     = Redirect{}
	_ Render     = Data{}
	_ Render     = HTML{}
	_ HTMLRender = HTMLDebug{}
	_ HTMLRender = HTMLProduction{}
	_ Render     = YAML{}
	_ Render     = MsgPack{}
	_ Render     = Reader{}
)
```
