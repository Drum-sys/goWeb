# goWeb

## HTTP 重写
当用户发起GET、POST请求，会调用addrouter方法将handler方法注册到map-router当中， key：GET-/helllo， value：handler函数。Run对http.ListenAndServe进行了封装启动服务器。
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

## Context上下文设计
封装 Request 和 Response ，提供对 JSON、HTML 等返回类型的支持。Context随着每一个请求的产生而产生，除了对Request和Response封装简化对外接口以外，还可以添加中间件的支持，解析对应的动态路由参数，请求头的封装等等。所以设计一个良好的上下文是必要的。

Context结构体包含了http.ResponseWriter和*http.Request，另外提供了对 Method 和 Path 这两个常用属性的直接访问。提供了访问Query和PostForm参数的方法。提供了快速构造String/Data/JSON/HTML响应的方法
```go
type Context struct {
	// origin objects
	Path string // 请求的路径 /hello
	Method string //请求的方法 GET POST等
	// request info
	Req *http.Request
	Writer http.ResponseWriter
	// response info
	StatusCode int
}

// 创建Context实体
func NewContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Req: req,
		Writer: w,
		Method: req.Method,
		Path: req.URL.Path,
		Params: make(map[string]string),
		index: -1,
	}
}


func (c *Context) PostForm(key string) string {
	return c.Req.FormValue(key)
}

func (c *Context) Query(key string) string {
	return c.Req.URL.Query().Get(key)
}

func (c *Context) Status(code int) {
	c.StatusCode = code
	c.Writer.WriteHeader(code)
}

func (c *Context) SetHeader(key string, value string) {
	c.Writer.Header().Set(key, value)
}

func (c *Context) String(code int, format string, values ...interface{}) {
	c.SetHeader("Content-Type", "text/plain")
	c.Status(code)
	c.Writer.Write([]byte(fmt.Sprintf(format, values...)))
}

func (c *Context) JSON(code int, obj interface{}) {
	c.SetHeader("Content-Type", "application/json")
	c.Status(code)
	encoder := json.NewEncoder(c.Writer)
	if err := encoder.Encode(obj); err != nil {
		http.Error(c.Writer, err.Error(), 500)
	}
}

func (c *Context) Data(code int, data []byte) {
	c.Status(code)
	c.Writer.Write(data)
}

func (c *Context) HTML(code int, html string) {
	c.SetHeader("Content-Type", "text/html")
	c.Status(code)
	c.Writer.Write([]byte(html))
}
```

将router独立出来方便以后增强， 将request和response替换为Context
```go

// 将router独立出来方便以后增强
type Router struct {
	handlers map[string]HandlerFunc // 路由的处理函数
}

type Engine struct {
  router *Router
}

// 创建Router对象
func NewRouter() *Router {
	return &Router{handlers: make(map[string]HandlerFunc),	
	}
}

// 添加路由
func (r *Router) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	r.handlers[key] = handler
}

func (r *router) handle(c *Context) {
	key := c.Method + "-" + c.Path
	if handler, ok := r.handlers[key]; ok {
		handler(c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := newContext(w, req)
	engine.router.handle(c)
}

```

