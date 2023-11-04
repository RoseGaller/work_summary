# motan-go源码分析

# 服务提供方

```go
func main() {
	runServerDemo()
}

func runServerDemo() {
 //加载配置信息
	mscontext := motan.GetMotanServerContext("serverdemo.yaml")
  //注册对外暴露的服务
	mscontext.RegisterService(&MotanDemoService{}, "")
  //暴露服务
	mscontex.Start(nil)
	time.Sleep(time.Second * 50000000)
}
//对外暴露的服务
type MotanDemoService struct{}
func (m *MotanDemoService) Hello(name string) string {
	fmt.Printf("MotanDemoService hello:%s\n", name)
	return "hello " + name
}
```

## 创建MSContext

```go
func GetMotanServerContext(confFile string) *MSContext {
	if !flag.Parsed() {
		flag.Parse()
	}
	serverContextMutex.Lock()
	defer serverContextMutex.Unlock()
	ms := serverContextMap[confFile] //根据配置文件获取对应的MSContext
	if ms == nil {

		ms = &MSContext{confFile: confFile}
		serverContextMap[confFile] = ms

		motan.Initialize(ms) //初始化MSContext
		section, err := ms.context.Config.GetSection("motan-server")
		if err != nil {
			fmt.Println("get config of \"motan-server\" fail! err " + err.Error())
		}

		logdir := ""
		if section != nil && section["log_dir"] != nil {
			logdir = section["log_dir"].(string)
		}
		if logdir == "" {
			logdir = "."
		}
		initLog(logdir)
	}
	return ms
}
```

### 初始化MSContext

core/motan.go

```go
func Initialize(s interface{}) {
  //MSContext实现了接口Initializable的Initialize方法
	if init, ok := s.(Initializable); ok {
		init.Initialize()
	}
}

```

```go
func (m *MSContext) Initialize() {
	m.csync.Lock()
	defer m.csync.Unlock()
	if !m.inited {
    //创建Context
		m.context = &motan.Context{ConfigFile: m.confFile}
		m.context.Initialize()

		m.portService = make(map[int]motan.Exporter, 32)
		m.portServer = make(map[int]motan.Server, 32)
		m.serviceImpls = make(map[string]interface{}, 32)
		m.inited = true
	}
}
```

#### 初始化Context

core/globalContext.go

```go
func (c *Context) Initialize() {
	c.RegistryUrls = make(map[string]*Url)
	c.RefersUrls = make(map[string]*Url)
	c.BasicRefers = make(map[string]*Url)
	c.ServiceUrls = make(map[string]*Url)
	c.BasicServiceUrls = make(map[string]*Url)
	if c.ConfigFile == "" { // 使用默认的配置文件
		c.ConfigFile = *CfgFile
	}
	//解析配置文件
	cfgRs, _ := cfg.NewConfigFromFile(c.ConfigFile)
	c.Config = cfgRs
  //解析配置的server、registry、basicService、service
	c.parseRegistrys()
	c.parseBasicRefers()
	c.parseRefers()
	c.parserBasicServices()
	c.parseServices()
	c.parseHostUrl()
}
```

## 注册服务

server.go

```go
func (m *MSContext) RegisterService(s interface{}, sid string) error {
	if s == nil {
		vlog.Errorln("MSContext register service is nil!")
		return errors.New("register service is nil!")
	}
	v := reflect.ValueOf(s)
	if v.Kind() != reflect.Ptr {
		vlog.Errorf("register service must be a pointer of struct. service:%+v\n", s)
		return errors.New("register service must be a pointer of struct!")
	}
	t := v.Elem().Type()
	hasConfig := false
	ref := sid
	if ref == "" {
		ref = t.String() //packageName.structName
	}
	// 检测配置文件是否有此服务的配置
	for _, url := range m.context.ServiceUrls {
		if url.Parameters != nil && ref == url.Parameters[motan.RefKey] {
			hasConfig = true
			break
		}
	}
	if !hasConfig { //无配置
		vlog.Errorf("can not find export config for register service. service:%+v\n", s)
		return errors.New("can not find export config for register service!")
	}
  //serviceId -> serviceImpl  
	m.serviceImpls[ref] = s
	return nil
}
```

## 暴露服务

```go
	func (m *MSContext) Start(extfactory motan.ExtentionFactory) {
		m.csync.Lock()
		defer m.csync.Unlock()
		m.extFactory = extfactory
		if m.extFactory == nil {
			m.extFactory = GetDefaultExtFactory()
		}
	
		for _, url := range m.context.ServiceUrls {
			m.export(url)
		}
	}
```

### 创建扩展工厂

default.go

```go
var (
	once              sync.Once
  //使用map存放filterFactories、haFactories、lbFactories、、、
	defaultExtFactory *motan.DefaultExtentionFactory
)
```

```go
func GetDefaultExtFactory() motan.ExtentionFactory {
  once.Do(func() { //只执行一次
    //实现了方法Initialize
    defaultExtFactory = &motan.DefaultExtentionFactory{}
    //初始化DefaultExtentionFactory
    defaultExtFactory.Initialize()
    //注册方法
    AddDefaultExt(defaultExtFactory)
  })
  return defaultExtFactory
}
```

#### 初始化

motan.go

```go
func (d *DefaultExtentionFactory) Initialize() {
  //初始化数据结构
	d.filterFactories = make(map[string]DefaultFilterFunc)
	d.haFactories = make(map[string]NewHaFunc)
	d.lbFactories = make(map[string]NewLbFunc)
	d.endpointFactories = make(map[string]NewEndpointFunc)
	d.providerFactories = make(map[string]NewProviderFunc)
	d.registryFactories = make(map[string]NewRegistryFunc)
	d.servers = make(map[string]NewServerFunc)
	d.registries = make(map[string]Registry)
	d.messageHandlers = make(map[string]NewMessageHandlerFunc)
	d.serializations = make(map[string]NewSerializationFunc)
}
```

#### 注册

```go
func AddDefaultExt(d motan.ExtentionFactory) {
  //注册默认功能实现
  filter.RegistDefaultFilters(d)
  ha.RegistDefaultHa(d)
  lb.RegistDefaultLb(d)
  endpoint.RegistDefaultEndpoint(d)
  provider.RegistDefaultProvider(d)
  registry.RegistDefaultRegistry(d)
  server.RegistDefaultServers(d)
  server.RegistDefaultMessageHandlers(d)
  serialize.RegistDefaultSerializations(d)
}
```

### 暴露服务

server.go

```go
func (m *MSContext) export(url *motan.Url) {
	defer func() {
		if err := recover(); err != nil {
			vlog.Errorf("MSContext export fail! url: %v, err:%v\n", url, err)
		}
	}()
  //获取服务实例
	service := m.serviceImpls[url.Parameters[motan.RefKey]]
	if service != nil {
    //例如motan2:8100
		export := url.GetParam(motan.ExportKey, "")
    //暴露端口
		port := defaultServerPort 
    //暴露协议
		protocol := defaultProtocal
		if export != "" {
			s := strings.Split(export, ":")
			if len(s) == 1 {
				port = s[0]
			} else if len(s) == 2 {
				if s[0] != "" {
					protocol = s[0]
				}
				port = s[1]
			}
		}
		url.Protocol = protocol
    //转换成int
		porti, err := strconv.Atoi(port)
		if err != nil {
			vlog.Errorf("export port not int. port:%s, url:%+v\n", port, url)
			return
		}
		url.Port = porti
		if url.Host == "" {
			url.Host = motan.GetLocalIp()
		}
    //默认DefaultProvider
    provider := GetDefaultExtFactory().GetProvider(url)
		provider.SetService(service)
    //初始化，维护服务实例的方法
		motan.Initialize(provider)
    //通过过滤器包装provider
		provider = mserver.WarperWithFilter(provider, m.extFactory)

    //创建exporter
		exporter := &mserver.DefaultExporter{}
		exporter.SetProvider(provider)
		//创建server
		server := m.portServer[url.Port]
		if server == nil {
      //根据协议获取server，MotanServer
			server = m.extFactory.GetServer(url)
      //DefaultMessageHandler
			handler := GetDefaultExtFactory().GetMessageHandler("default")
      //设置provider到DefaultMessageHandler
			motan.Initialize(handler)
			handler.AddProvider(provider)
      //绑定端口，接收外部请求
			server.Open(false, false, handler, m.extFactory)
			m.portServer[url.Port] = server
		} else if canShareChannel(*url, *server.GetUrl()) {
			server.GetMessageHandler().AddProvider(provider)
		} else {
			vlog.Errorf("service export fail! can not share channel.url:%v, port url:%v\n", url, server.GetUrl())
			return
		}
    //将服务注册到服务注册中心
		err = exporter.Export(server, m.extFactory, m.context)
		if err != nil {
			vlog.Errorf("service export fail! url:%v, err:%v\n", url, err)
		} else {
			vlog.Infof("service export success. url:%v\n", url)
		}
	}
}
```

motan.go

```go
func (d *DefaultExtentionFactory) GetProvider(url *Url) Provider {
	if newProviderFunc, ok := d.providerFactories[url.GetParam(ProviderKey, "default")]; ok {
		provider := newProviderFunc(url) //默认DefaultProvider
		return provider
	}
	vlog.Errorf("provider(protocol) name %s is not found in DefaultExtentionFactory!\n", url.Protocol)
	return nil
}

```

provider/provider.go

```go
func (d *DefaultProvider) Initialize() {
	d.methods = make(map[string]reflect.Value, 32)
	if d.service != nil && d.url != nil { //服务实例和url不能为空
		v := reflect.ValueOf(d.service)
		if v.Kind() != reflect.Ptr { //不是指针
			vlog.Errorf("can not init provider. service is not a pointer. service :%v, url:%v\n", d.service, d.url)
			return
		}
    //注册方法
		for i := 0; i < v.NumMethod(); i++ {
			name := v.Type().Method(i).Name
			vm := v.MethodByName(name)
			d.methods[name] = vm
		}

	} else {
		vlog.Errorf("can not init provider. service :%v, url:%v\n", d.service, d.url)
	}
}
```

```go
func WarperWithFilter(provider motan.Provider, extFactory motan.ExtentionFactory) motan.Provider {//包装Provider 过滤器链
	var lastf motan.EndPointFilter
	lastf = motan.GetLastEndPointFilter()
	_, filters := motan.GetUrlFilters(provider.GetUrl(), extFactory)
	for _, f := range filters {
		if ef, ok := f.NewFilter(provider.GetUrl()).(motan.EndPointFilter); ok {
			ef.SetNext(lastf)
			lastf = ef
		}
	}
	return &FilterProviderWarper{provider: provider, filter: lastf}
}
```

#### 接收请求

server/motanserver.go

```go
func (m *MotanServer) Open(block bool, proxy bool, handler motan.MessageHandler, extFactory motan.ExtentionFactory) error {
  //监听端口
	lis, err := net.Listen("tcp", ":"+strconv.Itoa(int(m.Url.Port)))
	if err != nil {
		vlog.Errorf("listen port:%d fail. err: %v\n", m.Url.Port, err)
		return err
	}
	m.listener = lis
	m.handler = handler //处理请求的入口
	m.extFactory = extFactory
	m.proxy = proxy
  //默认SimpleSerialization
	m.serialization = motan.GetSerialization(m.Url, m.extFactory)
	vlog.Infof("motan server is started. port:%d\n", m.Url.Port)
	if block {
		m.run()
	} else {
		go m.run()
	}
	return nil
}
```

```go
func (m *MotanServer) run() {
	for {
		conn, err := m.listener.Accept()
		if err != nil {
			vlog.Errorf("motan server accept from port %v fail. err:%s\n", m.listener.Addr(), err.Error())
		} else {

			go m.handleConn(conn)
		}
	}
}
```

```go
func (m *MotanServer) handleConn(conn net.Conn) {
	defer func() {
		if err := recover(); err != nil {
			vlog.Errorln("connection encount error! ", err)
		}
		conn.Close()
	}()
  //建立读缓冲区
	buf := bufio.NewReader(conn)
	for {
    	//读取请求
     request, err := mpro.DecodeFromReader(buf)
		if err != nil {
			if err.Error() != "EOF" {
				vlog.Warningf("decode motan message fail! con:%s\n.", conn.RemoteAddr().String())
			}
			break
		}
    //处理请求
		go m.processReq(request, conn)
	}
}
```

#### 处理请求

```go
func (m *MotanServer) processReq(request *mpro.Message, conn net.Conn) {
	defer func() {
		if err := recover(); err != nil {
			vlog.Errorln("Motanserver processReq error! ", err)
		}
	}()
	request.Header.SetProxy(m.proxy)

	var res *mpro.Message
	if request.Header.IsHeartbeat() { //心跳请求
		res = mpro.BuildHeartbeat(request.Header.RequestId, mpro.Res)
	} else {
		var mres motan.Response
    //根据请求头信息获取序列化方式
		serialization := m.extFactory.GetSerialization("", request.Header.GetSerialize())
		//将message转换成Request
    req, err := mpro.ConvertToRequest(request, serialization)
		req.GetRpcContext(true).ExtFactory = m.extFactory
		if err != nil {
			vlog.Errorf("motan server convert to motan request fail. rid :%d, service: %s, method:%s,err:%s\n", request.Header.RequestId, request.Metadata[mpro.M_path], request.Metadata[mpro.M_method], err.Error())
			mres = motan.BuildExceptionResponse(request.Header.RequestId, &motan.Exception{ErrCode: 500, ErrMsg: "deserialize fail. method:" + request.Metadata[mpro.M_method], ErrType: motan.ServiceException})
		} else {
      //DefaultMessageHandler
			mres = m.handler.Call(req) //处理请求
		}
		if mres != nil {
			mres.GetRpcContext(true).Proxy = m.proxy
      //序列化响应结果，创建Message
			res, err = mpro.ConvertToResMessage(mres, serialization)
		} else {
			err = errors.New("handler call return nil!")
		}

		if err != nil {
			res = mpro.BuildExceptionResponse(request.Header.RequestId, mpro.ExceptionToJson(&motan.Exception{ErrCode: 500, ErrMsg: "convert to response fail.", ErrType: motan.ServiceException}))
		}
	}
  //编码
	resbuf := res.Encode()
  //直接发送出去
	conn.Write(resbuf.Bytes())
}
```

protocol/motanProtocol.go

```go
func ConvertToRequest(request *Message, serialize motan.Serialization) (motan.Request, error) {
	motanRequest := &motan.MotanRequest{Arguments: make([]interface{}, 0)}
	motanRequest.RequestId = request.Header.RequestId
	motanRequest.ServiceName = request.Metadata[M_path]
	motanRequest.Method = request.Metadata[M_method]
	motanRequest.MethodDesc = request.Metadata[M_methodDesc]
	motanRequest.Attachment = request.Metadata
	rc := motanRequest.GetRpcContext(true)
	rc.OriginalMessage = request
	rc.Proxy = request.Header.IsProxy()
	if request.Body != nil && len(request.Body) > 0 {
		if request.Header.IsGzip() { //启用zip压缩
			request.Body = DecodeGzip	Body(request.Body) //解压缩
			request.Header.SetGzip(false)
		}
		if serialize == nil {
			return nil, errors.New("serialization is nil!")
		}
    //创建DeserializableValue，负责反序列化
		dv := &motan.DeserializableValue{Body: request.Body, Serialization: serialize}
		motanRequest.Arguments = []interface{}{dv}
	}

	return motanRequest, nil
}
```

server/server.go

```go
func (d *DefaultMessageHandler) Call(request motan.Request) (res motan.Response) {
	p := d.providers[request.GetServiceName()] //获取服务实例,FilterProviderWarper
	if p != nil {
		return p.Call(request) //执行请求
	} else { //服务实例不存在
		vlog.Errorf("not found provider for %s\n", motan.GetReqInfo(request))
		return motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: "not found provider for " + request.GetServiceName(), ErrType: motan.ServiceException})
	}
}
```

```go
func (f *FilterProviderWarper) Call(request motan.Request) (res motan.Response) {
	return f.filter.Filter(f.provider, request) //执行过滤器链，最终调用DefaultProvider
}
```

provider/provider.go

```go
func (d *DefaultProvider) Call(request motan.Request) (res motan.Response) {
	m, exit := d.methods[motan.FirstUpper(request.GetMethod())]
	if !exit {
		vlog.Errorf("mehtod not found in provider. %s\n", motan.GetReqInfo(request))
		return motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: "mehtod " + request.GetMethod() + " is not found in provider.", ErrType: motan.ServiceException})
	}
	defer func() {
		if err := recover(); err != nil {
			vlog.Errorf("provider call fail! e: %v, %s\n", err, motan.GetReqInfo(request))
			res = motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: fmt.Sprintf("request process fail in provider. e:%v", err), ErrType: motan.ServiceException})
		}
	}()
	//参数个数
	inNum := m.Type().NumIn()
	if inNum > 0 {
		values := make([]interface{}, 0, inNum)
		for i := 0; i < inNum; i++ {
			// TODO how to reflect value pointer???
			values = append(values, reflect.New(m.Type().In(i)).Type())
		}
    //反序列化参数
		err := request.ProcessDeserializable(values)
		if err != nil {
			return motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: "deserialize arguments fail." + err.Error(), ErrType: motan.ServiceException})
		}
	}
	//参数封装成Value类型
	vs := make([]reflect.Value, 0, len(request.GetArguments()))
	for _, arg := range request.GetArguments() {
		vs = append(vs, reflect.ValueOf(arg))
	}
	ret := m.Call(vs) //执行方法
	mres := &motan.MotanResponse{RequestId: request.GetRequestId()}
	if len(ret) > 0 { // 有返回值
		mres.Value = ret[0]
		res = mres
	}
	return res
}
```

core/motan.go

```go
func (m *MotanRequest) ProcessDeserializable(toTypes []interface{}) error {
  //反序列化请求体即请求参数
   if m.GetArguments() != nil && len(m.GetArguments()) == 1 {
      if d, ok := m.GetArguments()[0].(*DeserializableValue); ok {
         v, err := d.DeserializeMulti(toTypes)
         if err != nil {
            return err
         }
         m.SetArguments(v)
      }
   }
   return nil
}
```

```go
func (d *DeserializableValue) DeserializeMulti(v []interface{}) ([]interface{}, error) {
   return d.Serialization.DeSerializeMulti(d.Body, v)
}
```

serialize/simple.go

```go
func (s *SimpleSerialization) DeSerializeMulti(b []byte, v []interface{}) (ret []interface{}, err error) {
  //现在只支持一个参数
   var rv interface{}
   if v != nil {
      if len(v) == 0 {
         return nil, nil
      }
      if len(v) > 1 {
         return nil, errors.New("do not support multi value in SimpleSerialization")
      }
      rv, err = s.DeSerialize(b, v[0])
   } else {
      rv, err = s.DeSerialize(b, nil)
   }
   return []interface{}{rv}, err
}
```

```go
func (s *SimpleSerialization) DeSerialize(b []byte, v interface{}) (interface{}, error) {
   if len(b) == 0 {
      return nil, nil
   }
   buf := bytes.NewBuffer(b)
  //读取1个字节，即参数的类型
   tp, err := buf.ReadByte()
   if err != nil {
      return nil, err
   }
   switch tp {
   case 0:
      v = nil
      return nil, nil
   case 1:
      st, _, err := decodeString(buf)
      if err != nil {
         return nil, err
      }
      if v != nil {
         if sv, ok := v.(*string); ok {
            *sv = st
         }
      }
      return st, err
   case 2:
      ma, err := decodeMap(buf)
      if err != nil {
         return nil, err
      }
      if v != nil {
         if mv, ok := v.(*map[string]string); ok {
            *mv = ma
         }
      }
      return ma, err
   case 3:
      by, err := decodeBytes(buf)
      if err != nil {
         return nil, err
      }
      if v != nil {
         if bv, ok := v.(*[]byte); ok {
            *bv = by
         }
      }
      return by, err
   }
   return nil, errors.New(fmt.Sprintf("can not deserialize. unknown type:%v", tp))
}
```

#### 编码

##### ConvertToResMessage

protocol/motanProtocol.go

```go
func ConvertToResMessage(response motan.Response, serialize motan.Serialization) (*Message, error) { //将Response转换成Message
   rc := response.GetRpcContext(false)

   if rc != nil && rc.Proxy && rc.OriginalMessage != nil {
      if msg, ok := rc.OriginalMessage.(*Message); ok {
         msg.Header.SetProxy(true)
         return msg, nil
      }
   }

   res := &Message{Metadata: make(map[string]string)}
   var msgType int
   if response.GetException() != nil {
      msgType = Exception
      response.SetAttachment(M_exception, ExceptionToJson(response.GetException()))
   } else {
      msgType = Normal
   }
  //响应头（此message为响应消息、使用的序列化编码、对应的请求Id、响应是否正常）
   res.Header = BuildHeader(Res, false, serialize.GetSerialNum(), response.GetRequestId(), msgType)
   if response.GetValue() != nil { //有响应值
      if serialize == nil {
         return nil, errors.New("serialization is nil.")
      }
     	//序列化
      b, err := serialize.Serialize(response.GetValue())
      if err != nil {
         return nil, err
      }
      res.Body = b
   }
   res.Metadata = response.GetAttachments()

   if rc != nil {
      if rc.GzipSize > 0 && len(res.Body) > rc.GzipSize {
         data, err := EncodeGzip(res.Body) //压缩响应体
         if err != nil {
            vlog.Errorf("encode gzip fail! requestid:%d, err %s\n", response.GetRequestId(), err.Error())
         } else {
            res.Header.SetGzip(true)
            res.Body = data
         }
      }
      if rc.Proxy {
         res.Header.SetProxy(true)
      }
   }

   res.Header.SetSerialize(serialize.GetSerialNum())
   return res, nil
}
```

##### Serialize

serialize/simple.go

```go
func (s *SimpleSerialization) Serialize(v interface{}) ([]byte, error) {
  //序列化方法返回值
   if v == nil {
      return []byte{0}, nil
   }
  //获取方法返回值的类型（string/map[string]string/[]uint8）
   var rv reflect.Value
   if nrv, ok := v.(reflect.Value); ok {
      rv = nrv
   } else {
      rv = reflect.ValueOf(v)
   }
   t := fmt.Sprintf("%s", rv.Type())
   buf := new(bytes.Buffer)
  
   var err error
   switch t {
   case "string":
      buf.WriteByte(1) //类型
      _, err = encodeString(rv, buf) //返回值
   case "map[string]string":
      buf.WriteByte(2)
      err = encodeMap(rv, buf)
   case "[]uint8":
      buf.WriteByte(3)
      err = encodeBytes(rv, buf)
   }
   return buf.Bytes(), err
}
```

```go
func encodeString(v reflect.Value, buf *bytes.Buffer) (int32, error) {
   //编码字符串
   b := []byte(v.String())
   l := int32(len(b))
   err := binary.Write(buf, binary.BigEndian, l) //长度
   err = binary.Write(buf, binary.BigEndian, b) // 响应值
   if err != nil {
      return 0, err
   }
   return l + 4, nil
}
```

```go
func encodeMap(v reflect.Value, buf *bytes.Buffer) error {
  	//编码map[string]string
   b := new(bytes.Buffer) //存放map的数据
   var size, l int32
   var err error
   for _, mk := range v.MapKeys() {
      mv := v.MapIndex(mk)
      l, err = encodeString(mk, b)
      size += l
      if err != nil {
         return err
      }
      l, err = encodeString(mv, b)
      size += l
      if err != nil {
         return err
      }
   }
   err = binary.Write(buf, binary.BigEndian, int32(size)) //长度
   err = binary.Write(buf, binary.BigEndian, b.Bytes()[:size]) //值
   return err
}
```

```go
func encodeBytes(v reflect.Value, buf *bytes.Buffer) error {
   l := len(v.Bytes())
   err := binary.Write(buf, binary.BigEndian, int32(l))
   err = binary.Write(buf, binary.BigEndian, v.Bytes())
   return err
}
```

##### Encode

```go
func (msg *Message) Encode() (buf *bytes.Buffer) {
   //将message中的数据全部编码成字节
   buf = new(bytes.Buffer)
   var metabuf bytes.Buffer
   var meta map[string]string
   meta = msg.Metadata
   for k, v := range meta {
      metabuf.WriteString(k)
      metabuf.WriteString("\n")
      metabuf.WriteString(v)
      metabuf.WriteString("\n")
   }
   var metasize int32
   if metabuf.Len() > 0 {
      metasize = int32(metabuf.Len() - 1)
   } else {
      metasize = 0
   }

   body := msg.Body
   bodysize := int32(len(body))
  //header + meta长度 + meta + body长度 + body
   binary.Write(buf, binary.BigEndian, msg.Header) //header
   binary.Write(buf, binary.BigEndian, metasize) //meta长度
   if metasize > 0 {
      binary.Write(buf, binary.BigEndian, metabuf.Bytes()[:metasize]) //meta
   }

   binary.Write(buf, binary.BigEndian, bodysize) //body长度
   if bodysize > 0 { 
      binary.Write(buf, binary.BigEndian, body) //body
   }
   return buf
}
```

# 服务消费方

quickStart

```yaml
#config fo client
motan-client:
  mport: 8002 # client manage port
  log_dir: "./clientlogs" 
  application: "ray-test" # client identify.  

#config of registries
motan-registry:
  zk-registry:
  	 protocol: zookeeper
     host: 10.210.235.157
     port: 2181
     registrySessionTimeout: 10000
     requestTimeout: 5000 
  
#conf of basic refers
motan-basicRefer:
  mybasicRefer: # basic refer id
    group: motan-demo-rpc # group name
    protocol: motan2 # rpc protocol
    registry: "direct-registry" # registry id
    requestTimeout: 1000
    haStrategy: failover
    loadbalance: roundrobin
    serialization: simple
    filter: "accessLog" # filter registed in extFactory
    retries: 0

#conf of refers
motan-refer:
  mytest-motan2:
    path: com.weibo.motan2.test.Motan2TestService # e.g. service name for subscribe
    registry: zk-registry    
    basicRefer: mybasicRefer # basic refer id

```

```go
func main() {
   runClientDemo()
}

func runClientDemo() {
  //加载消费端的配置文件
   mccontext := motan.GetMotanClientContext("clientdemo.yaml")
   mccontext.Start(nil)
  //根据服务名称获取MotanClient
   mclient := mccontext.GetClient("mytest-motan2")

  //同步发送
   args := make(map[string]string, 16)
   args["name"] = "ray"
   args["id"] = "xxxx"
   var reply string
   err := mclient.Call("hello", args, &reply) //发送请求
   if err != nil {
      fmt.Printf("motan call fail! err:%v\n", err)
   } else {
      fmt.Printf("motan call success! reply:%s\n", reply)
   }
  	//异步发送
    args["key"] = "test async"
    result := mclient.Go("hello", args, &reply, make(chan *motancore.AsyncResult, 1))
    res := <-result.Done //等待响应
    if res.Error != nil {
      fmt.Printf("motan async call fail! err:%v\n", res.Error)
    } else {
      fmt.Printf("motan async call success! reply:%+v\n", reply)
    }
}
```

client.go

```java
func (m *MCContext) Start(extfactory motan.ExtentionFactory) {
   m.csync.Lock()
   defer m.csync.Unlock()
   m.extFactory = extfactory
   //获取DefaultExtentionFactory
   if m.extFactory == nil {
      m.extFactory = GetDefaultExtFactory()
   }

   for key, url := range m.context.RefersUrls { //motan-refer
     //MotanCluster	
      c := cluster.NewCluster(url, false)
      c.SetExtFactory(m.extFactory)
      c.Context = m.context
      c.InitCluster()
        //服务名称->MotanClient
      m.clients[key] = &MotanClient{url: url, cluster: c, extFactory: m.extFactory}
   }
}
```

## 初始化集群

cluster/motanCluster.go

```go
func (m *MotanCluster) InitCluster() bool {
   m.registryRefers = make(map[string][]motan.EndPoint)
   //HA FailOverHA
   m.HaStrategy = m.extFactory.GetHa(m.url)
   //lb RandomLB
   m.LoadBalance = m.extFactory.GetLB(m.url)
   //filter
   m.initFilters()
   // 解析注册中心并订阅
   m.parseRegistry()

   if m.clusterFilter == nil {
      m.clusterFilter = motan.GetLastClusterFilter()
   }
   if m.Filters == nil {
      m.Filters = make([]motan.Filter, 0)
   }
   //TODO weather has available refers
   m.available = true
   m.closed = false
   vlog.Infof("init MotanCluster %s\n", m.GetIdentity())

   return true
}
```

## 订阅服务

```go
func (m *MotanCluster) parseRegistry() (err error) {
  //direct-registry,zk-registry
   regs, ok := m.url.Parameters[motan.RegistryKey]
   if !ok {
      errInfo := fmt.Sprintf("registry not found! url %+v", m.url)
      err = errors.New(errInfo)
      vlog.Errorln(errInfo)
   }
   arr := strings.Split(regs, ",")
   registries := make([]motan.Registry, 0, len(arr))
   for _, r := range arr {
      if registryUrl, ok := m.Context.RegistryUrls[r]; ok {
        //ZkRegistry,DirectRegistry
         registry := m.extFactory.GetRegistry(registryUrl)
         if registry != nil {
            if _, ok := registry.(motan.DiscoverCommand); ok {
               registry = GetCommandRegistryWarper(m, registry)
            }
           	//订阅
            registry.Subscribe(m.url, m)
            registries = append(registries, registry)
           //发现服务
            urls := registry.Discover(m.url)
           // 触发通知
            m.Notify(registryUrl, urls)
         }
      } else {
         err = errors.New("registry is invalid: " + r)
         vlog.Errorln("registry is invalid: " + r)
      }

   }
   m.Registrys = registries
   return err
}
```

发现服务

registry/zkRegistry.go

```go
func (z *ZkRegistry) Discover(url *motan.Url) []*motan.Url {
   nodePath := ToNodeTypePath(url, ZOOKEEPER_NODETYPE_SERVER)
   if nodes, _, err := z.zkConn.Children(nodePath); err == nil {
      z.buildNodes(nodes, url)
      return buildUrl4Nodes(nodes, url)
   } else {
      vlog.Errorf("zookeeper registry discover fail! discover url:%s, err:%s\n", url.GetIdentity(), err.Error())
      return nil
   }
}
```

## 同步调用

client.go

```go
func (m *MotanClient) Call(method string, args interface{}, reply interface{}) error {
  //构建请求
   req := m.buildRequest(method, args)
   rc := req.GetRpcContext(true)
   rc.ExtFactory = m.extFactory
   rc.Reply = reply
   res := m.cluster.Call(req)
   if res.GetException() != nil {
      return errors.New(res.GetException().ErrMsg)
   } else {
      return nil
   }
}
```

### 构建请求

```go
func (m *MotanClient) buildRequest(method string, args interface{}) motan.Request {
   req := &motan.MotanRequest{Method: method, ServiceName: m.url.Path, Arguments: []interface{}{args}, Attachment: make(map[string]string, 16)}
   //附加信息
   version := m.url.GetParam(motan.VersionKey, "")
   if version != "" {
      req.Attachment[mpro.M_version] = version
   }
   module := m.url.GetParam(motan.ModuleKey, "")
   if module != "" {
      req.Attachment[mpro.M_module] = module
   }
   application := m.url.GetParam(motan.ApplicationKey, "")
   if application != "" {
      req.Attachment[mpro.M_source] = application
   }
   req.Attachment[mpro.M_group] = m.url.Group

   return req
}
```

### 发起调用

cluster/motanCluster.go

```go
func (m *MotanCluster) Call(request motan.Request) motan.Response {
   if m.available {
      return m.clusterFilter.Filter(m.HaStrategy, m.LoadBalance, request)
   } else {
      vlog.Infoln("cluster:" + m.GetIdentity() + "is not available!")
      return motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: "service cluster not available. maybe caused by degrade.", ErrType: motan.ServiceException})
   }

}
```

#### Filter

filter/clusterMetrics.go

```go
func (c *ClusterMetricsFilter) Filter(haStrategy motan.HaStrategy, loadBalance motan.LoadBalance, request motan.Request) motan.Response {
  //度量信息
   start := time.Now()
	 //执行下一个过滤器
   response := c.GetNext().Filter(haStrategy, loadBalance, request)

   m_p := strings.Replace(request.GetAttachments()["M_p"], ".", "_", -1)
   key := fmt.Sprintf("%s:%s.cluster:%s:%s", request.GetAttachments()["M_s"], request.GetAttachments()["M_g"], m_p, request.GetMethod())
   keyCount := key + ".total_count"
   metrics.AddCounter(keyCount, 1) //total_count

   if response.GetException() != nil { //err_count
      exception := response.GetException()
      if exception.ErrType == motan.BizException {
         bizErrCountKey := key + ".biz_error_count"
         metrics.AddCounter(bizErrCountKey, 1)
      } else {
         otherErrCountKey := key + ".other_error_count"
         metrics.AddCounter(otherErrCountKey, 1)
      }
   }

   end := time.Now()
   cost := end.Sub(start).Nanoseconds() / 1e6
   metrics.AddCounter((key + "." + metrics.ElapseTimeString(cost)), 1)

   if cost > 200 {
      metrics.AddCounter(key+".slow_count", 1)
   }

   metrics.AddHistograms(key, cost)
   return response
}
```

#### HA

core/motan.go

```go
func (l *lastClusterFilter) Filter(haStrategy HaStrategy, loadBalance LoadBalance, request Request) Response {
   return haStrategy.Call(request, loadBalance)
}
```

ha/failOverHA.go

```go
func (f *FailOverHA) Call(request motan.Request, loadBalance motan.LoadBalance) motan.Response {
   defer func() {
      if err := recover(); err != nil {
         vlog.Warningf("FailOverHA call encount panic! url:%s, err:%v\n", f.url.GetIdentity(), err)
      }
   }()
  	//获取重试次数
   retries := f.url.GetMethodPositiveIntValue(request.GetMethod(), request.GetMethodDesc(), "retries", defaultRetries)
   var lastErr *motan.Exception
   for i := 0; i <= int(retries); i++ {
     //挑选服务提供方
      ep := loadBalance.Select(request)
      if ep == nil {
         return getErrorResponse(request.GetRequestId(), fmt.Sprintf("No referers for request, RequestID: %d, Request info: %+v",
            request.GetRequestId(), request.GetAttachments()))
      }
      //发送请求
      respnose := ep.Call(request)
      if respnose.GetException() == nil || respnose.GetException().ErrType == motan.BizException {
         return respnose
      }
      lastErr = respnose.GetException()
      vlog.Warningf("FailOverHA call fail! url:%s, err:%+v\n", f.url.GetIdentity(), lastErr)
   }
   return getErrorResponse(request.GetRequestId(), fmt.Sprintf("call fail over %d times.Exception:%s", retries, lastErr.ErrMsg))

}
```

#### LB

挑选服务提供方

lb/randomLB.go

```go
func (r *RandomLB) Select(request motan.Request) motan.EndPoint {
   index := r.selectIndex(request)
   if index == -1 {
      return nil
   }
   return r.endpoints[index]
}
```

#### Endponit

endpoint/motanEndpoint.go

```go
func (m *MotanEndpoint) Call(request motan.Request) motan.Response {//发送请求
   rc := request.GetRpcContext(true)
   rc.Proxy = m.proxy
   if m.channels == nil {
      vlog.Errorln("motanEndpoint error: channels is null")
      m.recordErrAndKeepalive()
      return m.defaultErrMotanResponse(request, "motanEndpoint error: channels is null")
   }
   startTime := time.Now().UnixNano()
   if rc != nil && rc.AsyncCall {
      rc.Result.StartTime = startTime
   }
   //获取channel
   channel, err := m.channels.Get()
   if err != nil {
      vlog.Errorln("motanEndpoint error: can not get a channel.", err)
      m.recordErrAndKeepalive()
      return m.defaultErrMotanResponse(request, "can not get a channel")
   }
   // 请求超时时间
   deadline := m.url.GetTimeDuration("requestTimeout", time.Millisecond, defaultRequestTimeout)

   // 获取group
   group := GetRequestGroup(request)
   if group != m.url.Group && m.url.Group != "" {
      request.SetAttachment(mpro.M_group, m.url.Group)
   }
	//序列化请求
   var msg *mpro.Message
   msg, err = mpro.ConvertToReqMessage(request, m.serialization)

   if err != nil {
      vlog.Errorf("convert motan request fail! %s, err:%s\n", motan.GetReqInfo(request), err.Error())
      return motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: "convert motan request fail!", ErrType: motan.ServiceException})
   }
  //发送请求
   recvMsg, err := channel.Call(msg, deadline, rc)
   if err != nil {
      vlog.Errorln("motanEndpoint error: ", err)
      m.recordErrAndKeepalive()
      return m.defaultErrMotanResponse(request, "channel call error:"+err.Error())
   } else {
      // reset errorCount
      m.resetErr()
   }
   if rc != nil && rc.AsyncCall {
      return defaultAsyncResonse
   } else {
      recvMsg.Header.SetProxy(m.proxy)
     	//将响应信息封装成Response
      response, err := mpro.ConvertToResponse(recvMsg, m.serialization)
      if err != nil {
         vlog.Errorf("convert to response fail.%s, err:%s\n", motan.GetReqInfo(request), err.Error())
         return motan.BuildExceptionResponse(request.GetRequestId(), &motan.Exception{ErrCode: 500, ErrMsg: "convert response fail!" + err.Error(), ErrType: motan.ServiceException})

      }
     //将响应体内容反序列成reply类型
      response.ProcessDeserializable(rc.Reply)
      response.SetProcessTime(int64((time.Now().UnixNano() - startTime) / 1000000))
      return response
   }

}
```

##### 获取Channel

endpoint/motanEndpoint.go

```go
func (c *ChannelPool) Get() (*Channel, error) {
   channels := c.getChannels() //网络channel池子
   if channels == nil {
      return nil, errors.New("channels is nil")
   }
  //从池子获取网络channel
   channel, ok := <-channels
   if ok && (channel == nil || channel.IsClosed()) { //创建连接con
      conn, err := c.factory()
      if err != nil {
         vlog.Errorln("create channel failed.", err)
      }
      channel = buildChannel(conn, c.config) //创建网络channel
   }
   //将网络channel放回池子
   if err := retChannelPool(channels, channel); err != nil && channel != nil {
      channel.closeOnErr(err)
   }
   if channel == nil {
      return nil, errors.New("channel is nil")
   } else {
      return channel, nil
   }
}
```

###### 创建Channel

```go
func buildChannel(conn net.Conn, config *Config) *Channel { //创建网络channel
   if conn == nil {
      return nil
   }
   if config == nil {
      config = DefaultConfig()
   }
   if err := VerifyConfig(config); err != nil {
      return nil
   }
   channel := &Channel{
      conn:       conn,
      config:     config,
      bufRead:    bufio.NewReader(conn),
      sendCh:     make(chan sendReady, 64),
      streams:    make(map[uint64]*Stream, 64),
      heartbeats: make(map[uint64]*Stream),
      shutdownCh: make(chan struct{}),
   }

   go channel.recv() //接收响应

   go channel.send() //发送请求

   return channel
}
```

###### 发送请求

```go
func (c *Channel) send() {
   for {
      select {
      case ready := <-c.sendCh: //sendCh存放消费端需要发送的请求
         if ready.data != nil {
            sent := 0
            for sent < len(ready.data) {
              //同一个Channel上的请求顺序发送
               n, err := c.conn.Write(ready.data[sent:])
               if err != nil {
                  vlog.Errorf("Failed to write header: %v", err)
                  c.closeOnErr(err)
                  return
               }
               sent += n
            }
         }
      case <-c.shutdownCh:
         return
      }
   }
}
```

###### 接收响应

```go
func (c *Channel) recv() {
   if err := c.recvLoop(); err != nil {
      c.closeOnErr(fmt.Errorf("%+v", err))
   }
}
```

```go
func (c *Channel) recvLoop() error {
   for {
     //读取网络数据
      res, err := mpro.DecodeFromReader(c.bufRead)
      if err != nil {
         return err
      }
      var handleErr error
      if res.Header.IsHeartbeat() { //心跳响应信息
         handleErr = c.handleHeartbeat(res)
      } else { //非心跳响应信息
         handleErr = c.handleMessage(res)
      }	
      if handleErr != nil {
         return handleErr
      }
   }
}
```

###### 读取数据解码

```go
func DecodeFromReader(buf *bufio.Reader) (msg *Message, err error) {
   header := &Header{}
   //读取结构化二进制数据填充Header
   err = binary.Read(buf, binary.BigEndian, header)
   if err != nil {
      return nil, err
   }
	// 读取meta长度
   metasize := readInt32(buf)
   metamap := make(map[string]string)
   if metasize > 0 {
     //读取meta内容
      metadata, err := readBytes(buf, int(metasize))
      if err != nil {
         return nil, err
      }
      //meta内容，key、value按\n分割
      values := strings.Split(string(metadata), "\n")
      for i := 0; i < len(values); i++ {
         key := values[i]
         i++
         metamap[key] = values[i]
      }

   }
    //body长度
   bodysize := readInt32(buf)
   body := make([]byte, bodysize)
  //读取body内容
   if bodysize > 0 {
      body, err = readBytes(buf, int(bodysize))
   }
   if err != nil {
      return nil, err
   }
  //Message封装响应信息
   msg = &Message{header, metamap, body, Req}
   return msg, err
}
```

```go
func readInt32(buf io.Reader) int32 {
   var i int32
   binary.Read(buf, binary.BigEndian, &i) //读取4个字节
   return i
}
```

```go
func readBytes(buf *bufio.Reader, size int) ([]byte, error) {
   tempbytes := make([]byte, size)
   var s, n int = 0, 0
   var err error
   for s < size && err == nil {
      n, err = buf.Read(tempbytes[s:])//读取数据填充到byte数组
      s += n
   }
   return tempbytes, err
}
```

###### 处理响应

```go
func (c *Channel) handleMessage(msg *mpro.Message) error {
   c.streamLock.Lock()
   //获取stream
   stream := c.streams[msg.Header.RequestId]
   c.streamLock.Unlock()
   if stream == nil {
      vlog.Warningln("handle recv message, missing stream: ", msg.Header.RequestId)
   } else {
      stream.notify(msg)
   }
   return nil
}
```

###### 通知同步异步请求

```go
func (s *Stream) notify(msg *mpro.Message) {
   defer func() {
      s.Close()
   }()
   if s.rc != nil && s.rc.AsyncCall { //异步请求
      msg.Header.SetProxy(s.rc.Proxy)
      result := s.rc.Result //发送异步请求时会创建result
      serialization := s.rc.ExtFactory.GetSerialization("", msg.Header.GetSerialize()) //SimpleSerialization
      response, err := mpro.ConvertToResponse(msg, serialization)
      if err != nil {
         vlog.Errorf("convert to response fail. requestid:%d, err:%s\n", msg.Header.RequestId, err.Error())
         result.Error = err
         result.Done <- result //标志返回响应
         return
      }
     //将响应信息设置到reply
      response.ProcessDeserializable(result.Reply)
      response.SetProcessTime(int64((time.Now().UnixNano() - result.StartTime) / 1000000))
        result.Done <- result //通知异步请求，已经接收到响应信息
      return
   } else { //同步请求
      s.recvLock.Lock()
      s.recvMsg = msg //设置响应信息
      s.recvLock.Unlock()
      s.recvNotifyCh <- struct{}{} //唤醒请求
   }
}
```

```go
func ConvertToReqMessage(request motan.Request, serialize motan.Serialization) (*Message, error) {//将请求转换成Message
   rc := request.GetRpcContext(false)
   if rc != nil && rc.Proxy && rc.OriginalMessage != nil {
      if msg, ok := rc.OriginalMessage.(*Message); ok {
         msg.Header.SetProxy(true)
         return msg, nil
      }
   }
	//构建Header
   haeder := BuildHeader(Req, false, serialize.GetSerialNum(), request.GetRequestId(), Normal)
  //创建Message
   req := &Message{Header: haeder, Metadata: make(map[string]string)}
	//序列化请求体
   if len(request.GetArguments()) > 0 {
      if serialize == nil {
         return nil, errors.New("serialization is nil.")
      }
      b, err := serialize.SerializeMulti(request.GetArguments())
      if err != nil {
         return nil, err
      }
      req.Body = b
   }
   req.Metadata = request.GetAttachments()
   if rc != nil {
     	//是否对请求体压缩
      if rc.GzipSize > 0 && len(req.Body) > rc.GzipSize {
         data, err := EncodeGzip(req.Body)
         if err != nil {
            vlog.Errorf("encode gzip fail! %s, err %s\n", motan.GetReqInfo(request), err.Error())
         } else {
            req.Header.SetGzip(true)
            req.Body = data
         }
      }
      if rc.Oneway {
         req.Header.SetOneWay(true)
      }
      if rc.Proxy {
         req.Header.SetProxy(true)
      }
   }
   req.Header.SetSerialize(serialize.GetSerialNum())
   req.Metadata[M_path] = request.GetServiceName()
   req.Metadata[M_method] = request.GetMethod()
   if request.GetAttachment(M_proxyProtocol) == "" {
      req.Metadata[M_proxyProtocol] = "motan2"
   }

   if request.GetMethodDesc() != "" {
      req.Metadata[M_methodDesc] = request.GetMethodDesc()
   }
   return req, nil

}
```

##### 发送远程请求

endpoint/motanEndpoint.go

```go
func (c *Channel) Call(msg *mpro.Message, deadline time.Duration, rc *motan.RpcContext) (*mpro.Message, error) {
   stream, err := c.NewStream(msg, rc) //每发送一个请求创建一个stream
   if err != nil {
      return nil, err
   }
   stream.SetDeadline(deadline)//设置超时时间
   if err := stream.Send(); err != nil { //放入channel的发送通道sendch
      return nil, err
   } else {
      if rc != nil && rc.AsyncCall { //异步请求
         return nil, nil
      } else {
         return stream.Recv() //同步等待响应
      } 
   }
}
```

###### 请求放入sendCh

endpoint/motanEndpoint.go

```go
func (s *Stream) Send() error {
   timer := time.NewTimer(s.deadline.Sub(time.Now()))//请求超时定时器
   defer timer.Stop()//关闭定时器

   s.sendMsg.Header.RequestId = s.localRequestId
   buf := s.sendMsg.Encode()
   s.sendMsg.Header.RequestId = s.originRequestId

   ready := sendReady{data: buf.Bytes()}
   select {
    //放入发送通道sendCh，channel启动协程将请求发送给服务提供方
   case s.channel.sendCh <- ready: 
      return nil
   case <-timer.C: //通道满了，请求超时
      return ErrRequestTimeout
   case <-s.channel.shutdownCh: //网络channel关闭
      return ErrChannelShutdown
   }
}
```

###### 同步阻塞等待响应

```go
func (s *Stream) Recv() (*mpro.Message, error) { //同步等待响应
   defer func() {
      s.Close()
   }()
  //请求超时定时器
   timer := time.NewTimer(s.deadline.Sub(time.Now()))
   //方法结束，关闭定时器
   defer timer.Stop()
   select {
    //channel启动协程接收服务提供方返回的响应信息
   case <-s.recvNotifyCh: //接收到响应信息
      s.recvLock.Lock()
      msg := s.recvMsg //返回的结果信息
      s.recvLock.Unlock()
      if msg == nil {
         return nil, errors.New("recv err: recvMsg is nil")
      }
      msg.Header.RequestId = s.originRequestId
      return msg, nil
   case <-timer.C: //请求超时
      return nil, ErrRequestTimeout
   case <-s.channel.shutdownCh: //网络channel关闭
      return nil, ErrChannelShutdown
   }
}
```

##### 创建Response

```go
func ConvertToResponse(response *Message, serialize motan.Serialization) (motan.Response, error) {
   mres := &motan.MotanResponse{}
   rc := mres.GetRpcContext(true)

   mres.RequestId = response.Header.RequestId
   if response.Header.GetStatus() == Normal && len(response.Body) > 0 {
      if response.Header.IsGzip() { //启用压缩
         response.Body = DecodeGzipBody(response.Body) //解压
         response.Header.SetGzip(false)
      }
      if serialize == nil {
         return nil, errors.New("serialization is nil.")
      }
     //用于反序列化响应体的内容
      dv := &motan.DeserializableValue{Body: response.Body, Serialization: serialize}
      mres.Value = dv
   }
  //返回异常
   if response.Header.GetStatus() == Exception && response.Metadata[M_exception] != "" {
      var exception *motan.Exception
      err := json.Unmarshal([]byte(response.Metadata[M_exception]), &exception)
      if err != nil {
         return nil, err
      }
      mres.Exception = exception
   }
   mres.Attachment = response.Metadata
   rc.OriginalMessage = response
   rc.Proxy = response.Header.IsProxy()
   return mres, nil		
}
```

```go
func (m *MotanResponse) ProcessDeserializable(toType interface{}) error {
   if m.GetValue() != nil { //反序列化
      if d, ok := m.GetValue().(*DeserializableValue); ok {
         v, err := d.Deserialize(toType)
         if err != nil {
            return err
         }
         m.Value = v
      }
   }
   return nil
}
```

## 异步调用

```go
func (m *MotanClient) Go(method string, args interface{}, reply interface{}, done chan *motan.AsyncResult) *motan.AsyncResult {
   req := m.buildRequest(method, args)
   //创建AsyncResult
   result := &motan.AsyncResult{}
   if done == nil || cap(done) == 0 {
      done = make(chan *motan.AsyncResult, 5)
   }
   result.Done = done
  //创建请求的RpcContext
   rc := req.GetRpcContext(true)
   rc.ExtFactory = m.extFactory
   rc.Result = result
   rc.AsyncCall = true
   rc.Result.Reply = reply
   res := m.cluster.Call(req) //发起远程调用
   if res.GetException() != nil {
      result.Error = errors.New(res.GetException().ErrMsg)
      result.Done <- result
   }
   return result //返回AsyncResult
}
```

SimpleSerialization

```go
func (s *SimpleSerialization) Serialize(v interface{}) ([]byte, error) {//序列化
   if v == nil {
      return []byte{0}, nil
   }
   var rv reflect.Value
   if nrv, ok := v.(reflect.Value); ok {
      rv = nrv
   } else {
      rv = reflect.ValueOf(v)
   }

   t := fmt.Sprintf("%s", rv.Type())
   buf := new(bytes.Buffer)
   var err error
   switch t {
   case "string":
      buf.WriteByte(1)
      _, err = encodeString(rv, buf)
   case "map[string]string":
      buf.WriteByte(2)
      err = encodeMap(rv, buf)
   case "[]uint8":
      buf.WriteByte(3)
      err = encodeBytes(rv, buf)
   }
   return buf.Bytes(), err
}
```

```go
func (s *SimpleSerialization) DeSerialize(b []byte, v interface{}) (interface{}, error) { //反序列化
   if len(b) == 0 {
      return nil, nil
   }
   buf := bytes.NewBuffer(b)
   tp, err := buf.ReadByte()
   if err != nil {
      return nil, err
   }
   switch tp {
   case 0:
      v = nil
      return nil, nil
   case 1:
      st, _, err := decodeString(buf)
      if err != nil {
         return nil, err
      }
      if v != nil {
         if sv, ok := v.(*string); ok {
            *sv = st
         }
      }
      return st, err
   case 2:
      ma, err := decodeMap(buf)
      if err != nil {
         return nil, err
      }
      if v != nil {
         if mv, ok := v.(*map[string]string); ok {
            *mv = ma
         }
      }
      return ma, err
   case 3:
      by, err := decodeBytes(buf)
      if err != nil {
         return nil, err
      }
      if v != nil {
         if bv, ok := v.(*[]byte); ok {
            *bv = by
         }
      }
      return by, err
   }
   return nil, errors.New(fmt.Sprintf("can not deserialize. unknown type:%v", tp))
}
```

