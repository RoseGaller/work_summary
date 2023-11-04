# Quickstart

```go
func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

# 流程

## createEngine	

gin/gin.go

```go
func Default() *Engine {
   debugPrintWARNINGDefault()
   engine := New()
   engine.Use(Logger(), Recovery())
   return engine
}
```

```go
func New() *Engine {
   debugPrintWARNINGNew()
   engine := &Engine{
      RouterGroup: RouterGroup{ //维护所有的url及对应的handler
         Handlers: nil,
         basePath: "/",
         root:     true,
      },
      FuncMap:                template.FuncMap{},
      RedirectTrailingSlash:  true,
      RedirectFixedPath:      false,
      HandleMethodNotAllowed: false,
      ForwardedByClientIP:    true,
      RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
      TrustedProxies:         []string{"0.0.0.0/0"},
      TrustedPlatform:        defaultPlatform,
      UseRawPath:             false,
      RemoveExtraSlash:       false,
      UnescapePathValues:     true,
      MaxMultipartMemory:     defaultMultipartMemory,
      trees:                  make(methodTrees, 0, 9),
      delims:                 render.Delims{Left: "{{", Right: "}}"},
      secureJSONPrefix:       "while(1);",
   }
   engine.RouterGroup.engine = engine
   engine.pool.New = func() interface{} { //对象池，创建Context
      return engine.allocateContext()
   }
   return engine
}
```

## bindHandler

gin/routergroup.go

```go
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
   return group.handle(http.MethodGet, relativePath, handlers)
}
```

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
   absolutePath := group.calculateAbsolutePath(relativePath)
   handlers = group.combineHandlers(handlers) //合并handlers
   group.engine.addRoute(httpMethod, absolutePath, handlers)
   return group.returnObj()
}
```

gin/gin.go

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
   assert1(path[0] == '/', "path must begin with '/'")
   assert1(method != "", "HTTP method can not be empty")
   assert1(len(handlers) > 0, "there must be at least one handler")

   debugPrintRoute(method, path, handlers)

   root := engine.trees.get(method)	//GET、HEAD、POST...
   if root == nil {
      root = new(node)
      root.fullPath = "/"
      //[]methodTree
      engine.trees = append(engine.trees, methodTree{method: method, root: root})
   }
  //维护path与handler的映射关系
   root.addRoute(path, handlers)

   // Update maxParams
   if paramsCount := countParams(path); paramsCount > engine.maxParams {
      engine.maxParams = paramsCount
   }
}
```

## startEngine

gin/gin.go

```go
func (engine *Engine) Run(addr ...string) (err error) {
   defer func() { debugPrintError(err) }()

   err = engine.parseTrustedProxies()
   if err != nil {
      return err
   }

   address := resolveAddress(addr)
   debugPrint("Listening and serving HTTP on %s\n", address)
  //engine实现了ServeHTTP方法
   err = http.ListenAndServe(address, engine) //默认监听8080
   return
}
```

### ServeHTTP

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {//接收http请求
  //从对象池获取Context
  c := engine.pool.Get().(*Context)
  //重置responseWriter
   c.writermem.reset(w)
   c.Request = req
   c.reset()

   engine.handleHTTPRequest(c)
	//归还对象
   engine.pool.Put(c)
}
```

gin/response_writer.go

```go
func (w *responseWriter) reset(writer http.ResponseWriter) {
   w.ResponseWriter = writer
   w.size = noWritten  // -1
   w.status = defaultStatus // 默认OK
}
```

### handleHTTPRequest

gin/gin.go

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
   httpMethod := c.Request.Method
   rPath := c.Request.URL.Path
   unescape := false
   if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
      rPath = c.Request.URL.RawPath
      unescape = engine.UnescapePathValues
   }

   if engine.RemoveExtraSlash {
      rPath = cleanPath(rPath)
   }

   // Find root of the tree for the given HTTP method
   t := engine.trees
   for i, tl := 0, len(t); i < tl; i++ {
      if t[i].method != httpMethod {
         continue
      }
      root := t[i].root
      // Find route in tree
      value := root.getValue(rPath, c.params, unescape)
      if value.params != nil {
         c.Params = *value.params
      }
      if value.handlers != nil {
         c.handlers = value.handlers
         c.fullPath = value.fullPath
         c.Next()
         c.writermem.WriteHeaderNow()
         return
      }
      if httpMethod != http.MethodConnect && rPath != "/" {
         if value.tsr && engine.RedirectTrailingSlash {
            redirectTrailingSlash(c)
            return
         }
         if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
            return
         }
      }
      break
   }

   if engine.HandleMethodNotAllowed {
      for _, tree := range engine.trees {
         if tree.method == httpMethod {
            continue
         }
         if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
            c.handlers = engine.allNoMethod
            serveError(c, http.StatusMethodNotAllowed, default405Body)
            return
         }
      }
   }
   c.handlers = engine.allNoRoute
   serveError(c, http.StatusNotFound, default404Body)
}
```

# 组件

## binding

## render

## gin

## RouterGroup

## tree