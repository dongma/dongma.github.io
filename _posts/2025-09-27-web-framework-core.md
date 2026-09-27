---
layout: post
title: net/http启动服务
date: 2025-09-27 +0800
tags: ["前端"]
---

<!-- more -->
### 示例：启动`http`服务
在`go`中启动一个`http`服务只需寥寥几笔，调用`http.HandleFunc`注册路由，同时启动`http.ListenAndServe`启动一个`8091`的`http`服务。
```go
import (
    "net/http"
)

func main() {
    http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("pong"))
    })
    http.ListenAndServe(":8091", nil)
}

// sendJsonData 以json格式发送请求
func sendJsonData() {
    reqBody, _ := json.Marshal(map[string]string{"key1": "val1", "key2": "val2"})
    resp, _ := http.Post(":8091", "application/json", bytes.NewReader(reqBody))
    defer resp.Body.Close()
    
    respBody, _ := io.ReadAll(resp.Body)
    fmt.Printf("resp: %s", respBody)
}
```
### 注册Handler逻辑
在构建`http`服务过程中，`server`启动涉及核心的数据结构有如下几个：
- `Server`：整个 http 服务端模块被封装在`Server`类中，`Handler`是`Server`中最核心的成员字段；
- `Handler`：`Handler`是一个`interface`，暴露了方法：`ServeHTTP`，根据 http 请求`Request`中的请求路径`path`映射到对应的`handler`处理函数，对请求进行处理和响应；
- `ServeMux`：是对`Handler`的具体实现，内部通过一个`map`维护了从`path`到`handler`的映射关系.
- `muxEntry`：`muxEntry`为一个`handler`单元，内部包含了请求路径`path` + 处理函数`handler`两部分。

在`net/http`包下声明了一个单例`ServeMux`，当用户通过方法`http.HandleFunc`注册`handler`时，则会将其注册到`DefaultServeMux`当中：
```go
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```
`ServeMux.HandleFunc`内部会将处理函数`handler`转为`HandlerFunc`类型，将其作为`Handler interface`的实现类注册到`ServeMux`的路由`map`当中，实现路由注册的核心逻辑位于`ServeMux.Handle`方法中：
- 将`path`和`handler`包装成一个`muxEntry`，以`path`为`key`注册到路由`map ServeMux.m`中;
- 响应模糊匹配机制. 对于以 '/' 结尾的`path`，根据`path`长度将`muxEntry`有序插入到数组`ServeMux.es`中；
```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    // ...
    mux.Handle(pattern, HandlerFunc(handler))
}
```
<div>
    <img src="https://picx.zhimg.com/v2-72f71b4b5ed1978d13a5973eae414e37_r.jpg" width="580"/>
</div>

### 启动Server逻辑
启动入口是`server.go#ListenAndServe` -> `server.go#Server.ListenAndServe` -> 创建监听`net.Listen("tcp", addr)` -> `Server#Serve`方法，其中`Server.Serve`很核心，`for` + `listener.accept`模式.
```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}

func (srv *Server) ListenAndServe() error {
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    return srv.Serve(ln)
}

func (srv *Server) Serve(l net.Listener) error {
    for {
        rw, err := l.Accept()
        connCtx := ctx
        c := srv.newConn(rw)
        go c.serve(connCtx)
    }
}

// conn.serve 从conn中读取封装到response结构体，根据url找到匹配的Handler，然后执行
func (c *conn) serve(ctx context.Context) {
    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)
	
    for {
        w, err := c.readRequest(ctx)
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
    }
}
```
<div>
    <img src="https://pica.zhimg.com/v2-75514b1cbd61ad66d69909b5867b517a_r.jpg" width="580">
</div>

`ServeMux.ServeHTTP`、`ServeMux.Handler`、`ServeMux.handler`等方法，最终在`ServeMux.match`方法中，以`Request`中的`path`为`pattern`，在路由字典`Server.m`中匹配`handler`，最后调用`handler.ServeHTTP`方法进行请求的处理和响应。
```go
// ServeMux.ServeHTTP -> ServeMux.Handler -> ServeMux.handler
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    return mux.handler(host, r.URL.Path)
}
// mux.match方法，未命中时，会开启模糊匹配（与当前url匹配的最长路径）
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()
    h, pattern = mux.match(path)
    return
}
```
<div>
    <img src="https://pic3.zhimg.com/v2-266bb70c56357579eb7807f64e92a96e_r.jpg" width="636">
</div>

### Gin
可以看出，在`net/http`的既定框架下，`gin`所做的是提供了一个`gin.Engine`对象作为`Handler`注入其中，从而实现路由注册/匹配、请求处理链路的优化。
<div>
    <img src="https://pic4.zhimg.com/v2-306593a0eeef2b37f7da021272aae7ad_r.jpg" width="580">
</div>

`Engine`为`Gin`中构建的`HTTP Handler`，其实现了`net/http`包下`Handler interface`的抽象方法： `Handler.ServeHTTP`，因此可以作为`Handler`注入到`net/http`的`Server`当中，`Engine`包含的核心内容包括：
- 路由组`RouterGroup`：第（2）部分展开
- `Context`对象池`pool`：基于`sync.Pool`实现，作为复用`gin.Context`实例的缓冲池. `gin.Context`的内容于本文第`5`章详解；
- 路由树数组`trees`：共有`9`棵路由树，对应于`9`种`http`方法. 路由树基于压缩前缀树实现，于本文第`4`章详解.
<div>
    <img src="https://picx.zhimg.com/v2-9f452cc2bcd08ab6444750d5b0e750dd_1440w.jpg" width="580">
</div>

### 注册`handler`
以`http post`为例，注册`handler`方法调用顺序为`RouterGroup.POST`->`RouterGroup.handle`，接下来会完成三个步骤：
- 拼接出待注册方法的完整路径`absolutePath`
- 拼接出代注册方法的完整处理函数链`handlers`
- 以`absolutePath`和`handlers`组成`kv`对添加到路由树中
```go
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodPost, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath)
    handlers = group.combineHandlers(handlers)
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
```
<div>
    <img src="https://pic4.zhimg.com/v2-10825ae29725230f61a5b9b1f89dcc59_1440w.jpg" width="580"/>
</div> 

### 处理请求
`Engine.handleHTTPRequest`方法核心步骤分为三步：
- 根据`http method`取得对应的 methodTree
- 根据`path`从`methodTree`中找到对应的`handlers`链
- 将`handlers`链注入到`gin.Context`中，通过`Context.Next`方法按照顺序遍历调用`handler`
此处根据`path`从路由树寻找`handlers`的逻辑位于`root.getValue`方法中，和路由树数据结构有关，放在本文第`4`章详解；
<div>
    <img src="https://pic1.zhimg.com/v2-9514b00586e9c7337c03c2e761549032_r.jpg" width="580"/>
</div> 

### 相关扩展阅读：
- [Golang Http标准库实现原理](https://zhuanlan.zhihu.com/p/609258171)
- [解析Gin框架底层原理](https://zhuanlan.zhihu.com/p/611116090)
- [golang netpoll底层原理](https://zhuanlan.zhihu.com/p/721422268)
- [Go语言设计与实现](https://draven.co/golang/)
- [Uber Go编码格式](https://github.com/xxjwxc/uber_go_guide_cn)
- [低配Spring—Golang IOC框架dig原理解析](https://zhuanlan.zhihu.com/p/613040547)
