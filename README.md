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

## 前缀树实现动态路由

参数匹配:---例如 /p/:lang/doc，可以匹配 /p/c/doc 和 /p/go/doc。

通配*----例如 /static/*filepath，可以匹配/static/fav.ico，也可以匹配/static/js/jQuery.js，这种模式常用于静态服务器，能够递归地匹配子路径。
![图片](https://user-images.githubusercontent.com/82791037/195329700-fda7687e-8acd-4474-bc1f-d5b4a75bd7b5.png)

定义节点node, matchChild用于插入时匹配节点， matchChildren用于查询时匹配节点， 都是递归算法
```go

type node struct {
	pattern string //待匹配的路由 
	part string //路由的一部分
	wildChild bool // 是否精确匹配， 判断是否含有 * || ：
	children []*node
}

//找到第一个匹配的节点
func (n *node) matchChild(part string) *node {
	for _, child := range n.children {
		if child.part == part || n.wildChild {
			return child
		}
	}
	return nil
}

//寻找所有匹配的节点， 用于查找
func (n *node) matchChildren(part string) []*node {
	var nodes []*node
	for _, child := range n.children {
		if child.part == part || n.wildChild {
			nodes = append(nodes, child)
		}
	}
	return nodes
}
````
insert--向前缀树中插入节点, 客户端发起请求，路径为pattern（/hello/:lang/ljw）， part为将pattern分割后的序列， height用来表示树的深度， 初始为0.

当匹配到叶子节点时， 即height == len(parts)， n.pattern才被设置为/hello/:lang/ljw。当匹配结束时，我们可以使用n.pattern == ""来判断路由规则是否匹配成功。例如，/p/python虽能成功匹配到:lang，但:lang的pattern值为空

递归的遍历每一层节点，如果没有匹配到当前part的节点，则新建一个。

查询功能，同样也是递归查询每一层的节点，退出规则是，匹配到了*，匹配失败，或者匹配到了第len(parts)层节点
```go
// 插入节点
func (n *node) insert(pattern string, parts []string, height int) {
	if height == len(parts) {
		n.pattern = pattern
		//fmt.Println("insert children", n.pattern)
		return
	}

	part := parts[height]
	child := n.matchChild(part)

	if child == nil {
		child = &node{
			part: part,
			wildChild: part[0] == ':' || part[0] == '*',
		}
		n.children = append(n.children, child)
	}
	child.insert(pattern, parts, height+1)
}

// 查找匹配的节点
func (n *node) search(parts []string, height int) *node {
	if height == len(parts) || strings.HasPrefix(n.part, "*") {
		if n.pattern == "" {
			return nil
		}
		return n
	}

	part := parts[height]
	children := n.matchChildren(part)

	for _, child := range children {
		//递归的查询
		result := child.search(parts, height+1)
		if result != nil {
			return result
		}
	}
	return nil

}
```

将前缀树应用到router中， Router结构体中添加root， key: GET-/hello, value: node

修改addRoute方法， 将请求的pattern存储到前缀树当中，parsePatten负责解析pattern， 遇到*就结束（*出现在最后）。addrouter中调用insert方法想树中插入节点。


```go
type Router struct {
	handlers map[string]HandlerFunc // 路由的处理函数
	roots    map[string]*node // 路由的trie节点
}

func parsePatten(pattern string) []string {
	v := strings.Split(pattern, "/")

	parts := make([]string, 0)
	for _, part := range v {
		if part != "" {
			parts = append(parts, part)
			if part[0] == '*' {
				break
			}
		}
	}
	return parts
}

func (r *Router) addRoute(method string, pattern string, handler HandlerFunc) {
	parts := parsePatten(pattern)
	_, ok := r.roots[method]
	if !ok {
		r.roots[method] = &node{}
	}
	key := method + "-" + pattern
	r.roots[method].insert(pattern, parts, 0)
	r.handlers[key] = handler
}
```

getRoute负责解析路由参数（包括两种通配符 * ， ：）， 并返回一个map。例如/p/go/doc匹配到/p/:lang/doc，解析结果为：{lang: "go"}，/static/css/geektutu.css匹配到/static/*filepath，解析结果为{filepath: "css/geektutu.css"}
```go
func (r *Router) getRoute(method string, path string) (*node, map[string]string) {
	//searchParts := strings.Split(path, "/")
	searchParts := parsePatten(path)
	params := make(map[string]string)
	root, ok := r.roots[method]
	if !ok {
		return nil, nil
	}
	n := root.search(searchParts, 0)
	if n != nil {
		parts := parsePatten(n.pattern)
		for index, part := range parts {
			if part[0] == ':' {
				params[part[1:]] = searchParts[index]
			}

			if part[0] == '*' {
				params[part[1:]] = strings.Join(searchParts[index:], "/")
				break
			}
		}
		return n, params
	}
	return nil, nil
}
```

在 HandlerFunc 中，希望能够访问到解析的参数，因此，需要对 Context 对象增加一个属性和方法，来提供对路由参数的访问。我们将解析后的参数存储到Params中，通过c.Param("lang")的方式获取到对应的值。

```go
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	Params map[string]string
	// response info
	StatusCode int
}

func (c *Context) Param(key string) string {
	value, _ := c.Params[key]
	return value
}
```

handle函数调用中getRoute解析参数，返回找到的叶子节点n， 以及将解析到的参数存储到Context中。
```go
func (r *router) handle(c *Context) {
	n, params := r.getRoute(c.Method, c.Path)
	if n != nil {
		c.Params = params
		key := c.Method + "-" + n.pattern
		r.handlers[key](c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
```

## 路由分组实现
如果没有路由分组需要对每一个路由单独控制， 但是真实得业务场景中，一组路由需要进行相似的处理。同时可以将中间件作用与不同的分组上，避免重复添加。

- 以/post开头的路由匿名可访问。
- 以/admin开头的路由需要鉴权。
- 以/api开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

一般路由分组是通过前缀区分的即prefix属性， RouterGroup需要添加路由则需要引入*Engine.addroute()。

Engine可以被认为是所有路由分组得祖先，自然需要拥有RouteGroup得所有权限，即引入*RouterGroup。同时需要管理所有分组即groups
```go
type RouterGroup struct {
	prefix      string        //路由分组前缀
	middlewares []HandlerFunc //应用于分组的中间件
	//parent      *RouterGroup  //子分组的父亲是谁
	engine      *Engine       // 原先是Engine实现添加路由等， 集成到RouterGroup，既可以分组添加路由，也可以单独添加路由
}

// Engine implement the interface of ServeHTTP
type Engine struct {
	*RouterGroup
	router *Router
	groups []*RouterGroup
}
```

初始化顺序是，先创建Engine对象， 在创建RouteGroup对象，添加分组
```go
// New is the constructor of gee.Engine
func New() *Engine {
	engine := &Engine{router: NewRouter()}
	engine.RouterGroup = &RouterGroup{engine: engine}
	engine.groups = []*RouterGroup{engine.RouterGroup}
	return engine
}

// Group is defined to create a new RouterGroup
// remember all groups share the same Engine instance
func (group *RouterGroup) Group(prefix string) *RouterGroup {
	engine := group.engine
	newGroup := &RouterGroup{
		engine: engine,
		middlewares: make([]HandlerFunc, 0),
		//parent: group,
		prefix: group.prefix + prefix, 
	}
	engine.groups = append(engine.groups, newGroup)
	return newGroup

}
```

增加RouterGroup得addRoute方法添加路由映射。匹配路径为分组前缀+请求路径
```go
func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
	pattern := group.prefix + comp
	group.engine.router.addRoute(method, pattern, handler)
}

func (group *RouterGroup) GET(pattern string, handler HandlerFunc) {
	group.addRoute("GET", pattern, handler)
}

func (group *RouterGroup) POST(pattern string, handler HandlerFunc) {
	group.addRoute("POST", pattern, handler)
}
```
## 中间件插入
中间件的定义与路由映射的 Handler 一致，处理的输入是Context对象。插入点是框架接收到请求初始化Context对象后，允许用户使用自己定义的中间件做一些额外的处理，例如记录日志等，以及对Context进行二次加工。另外通过调用(*Context).Next()函数，中间件可等待用户自己定义的 Handler处理结束后，再做一些额外的操作，例如计算本次处理所用时间等。即 Gee 的中间件支持用户在请求被处理的前后，做一些额外的操作。举个例子，我们希望最终能够支持如下定义的中间件，c.Next()表示等待执行其他的中间件或用户的Handler：

接收到请求后，应查找所有应作用于该路由的中间件，保存在Context中，依次进行调用。为什么依次调用后，还需要在Context中保存呢？因为在设计中，中间件不仅作用在处理流程前，也可以作用在处理流程后，即在用户定义的 Handler 处理完毕后，还可以执行剩下的操作。

为此，我们给Context添加了2个参数，定义了Next方法.NEXt方法也要是为了在处理请求结束后做一些事。 index记录执行到了第几个中间件，循环中的index++和 index++ 防止中间件重复使用。index也必须保存到context中，状态一致。
如果没有index++， 【A, B, Handler】 注册3个中间件
- index=0， 先执行A， 内部调用next（）此时index还是等于0， 并没有切换到B
```go
type Context struct {
	// origin objects
	Path string // 请求的路径 /hello
	Method string //请求的方法 GET POST等
	// request info
	Req *http.Request
	Writer http.ResponseWriter
	Params map[string]string //存储动态路由匹配的参数 /hello/:lang ---> lang : ljw
	// response info
	StatusCode int
	// middleware
	handlers []HandlerFunc
	index    int 
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Path:   req.URL.Path,
		Method: req.Method,
		Req:    req,
		Writer: w,
		index:  -1,
	}
}

func (c *Context) Next() {
	c.index++
	s := len(c.handlers)
	for ; c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}
```

将中间件应用到Group, 改变ServeHTTP，当我们接收到一个具体请求时，要判断该请求适用于哪些中间件，在这里我们简单通过 URL 的前缀来判断。得到中间件列表后，赋值给 c.handlers
```go
// Use is defined to add middleware to the group
func (group *RouterGroup) Use(middlewares ...HandlerFunc) {
	group.middlewares = append(group.middlewares, middlewares...)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	var middlewares []HandlerFunc
	for _, group := range engine.groups {
		if strings.HasPrefix(req.URL.Path, group.prefix) {
			middlewares = append(middlewares, group.middlewares...)
		}
	}
	c := newContext(w, req)
	c.handlers = middlewares
	engine.router.handle(c)
}
```
handle 函数中，将从路由匹配得到的 Handler 添加到 c.handlers列表中，执行c.Next()
```go
func (r *router) handle(c *Context) {
	n, params := r.getRoute(c.Method, c.Path)

	if n != nil {
		key := c.Method + "-" + n.pattern
		c.Params = params
		c.handlers = append(c.handlers, r.handlers[key])
	} else {
		c.handlers = append(c.handlers, func(c *Context) {
			c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
		})
	}
	c.Next()
}
```

