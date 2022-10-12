# goWeb

## HTTP 重写
```go

// HandlerFunc defines the request handler used by goweb
type HandlerFunc func(http.ResponseWriter, *http.Request)

// 定义Engine 实现http.handler接口中的ServeHTTP方法， 
type Engine struct {
  router map[string]HandlerFunc
}

//添加路由
func (r *Router) addRoute(method string, pattern string, handler HandlerFunc) {
  key := method + "-" + pattern // 类似 GET-/hello
  engine.router[key] = handler
}

// 客户端发起get请求
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

// 客户端发起POST请求
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

// Run 启动服务器
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}

// http.handler接口， 用户自定义handler方法
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {
		handler(w, req)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}
```

