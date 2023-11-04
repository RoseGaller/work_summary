# Go-ServiceMesher源码分析

* [前置初始化](#前置初始化)
  * [Protocol](#protocol)
    * [httpServer](#httpserver)
    * [grpcServer](#grpcserver)
    * [dubboServer](#dubboserver)
  * [Resolver](#resolver)
  * [Egress](#egress)
    * [CSE](#cse)
    * [Pilot](#pilot)
* [启动入口](#启动入口)
  * [CmdInit](#cmdinit)
  * [RegisterFramework](#registerframework)
  * [SetHandlers](#sethandlers)
  * [InitEgressChain](#initegresschain)
  * [BootstrapStart](#bootstrapstart)
    * [Config](#config)
    * [Resolver](#resolver-1)
    * [Egress](#egress-1)
    * [control](#control)
  * [Health](#health)
* [Http协议](#http协议)
  * [启动sidecar](#启动sidecar)
    * [LocalRequestHandler](#localrequesthandler)
    * [RemoteRequestHandler](#remoterequesthandler)


官网https://mesher.readthedocs.io/

# 前置初始化

## Protocol

### httpServer

proxy/protocol/http/http_server.go

```go
func init() {
   server.InstallPlugin(Name, newServer)//安装http服务
} 
```

```go
func newServer(opts server.Options) server.ProtocolServer {
   return &httpServer{ //创建Http服务
      opts: opts,
   }
}
```

### grpcServer

proxy/protocol/grpc/server.go:43

```go
func init() {
   server.InstallPlugin(Name, newServer //安装grpc服务
}
```

```go
func newServer(opts server.Options) server.ProtocolServer {
   return &httpServer{
      opts: opts,
   }
}
```

### dubboServer

proxy/protocol/dubbo/server/server.go

```go
func init() {
   server.InstallPlugin(NAME, newServer) //安装dubbo服务
}
```

```go
func newServer(opts server.Options) server.ProtocolServer {

   return &DubboServer{
      opts:       opts,
      routineMgr: util.NewRoutineManager(),
   }
}
```

## Resolver

proxy/resolver/destination.go

```go
const DefaultPlugin = "host"
```

```go
func init() {
   DestinationResolverPlugins = make(map[string]func() DestinationResolver)
   InstallDestinationResolverPlugin(DefaultPlugin, New) //name ->func
   SetDefaultDestinationResolver("http", &DefaultDestinationResolver{})
}
```

```go
func New() DestinationResolver {
   return &DefaultDestinationResolver{}
}
```

## Egress

### CSE

proxy/pkg/egress/archaius/egress.go

```go
func init() {
   egress.InstallEgressService("cse", newEgress)//默认
}
```

```go
func newEgress() (egress.Egress, error) {
   return &Egress{}, nil
}
```

### Pilot

proxy/pkg/egress/pilot/egress.go

```go
func init() {
  egress.InstallEgressService("pilot", newPilotEgress) 
}
```

```go
func newPilotEgress() (egress.Egress, error) {
  return &PilotEgress{}, nil 
}
```

# 启动入口

cmd/mesher/mesher.go

```go
func main() {
   server.Run()
}
```

proxy/server/server.go

```go
func Run() {
	//解析命令行参数
	if err := cmd.Init(); err != nil {
		panic(err)
	}
	if err := cmd.Configs.GeneratePortsMap(); err != nil {
		panic(err)
	}
	bootstrap.RegisterFramework()
	//设置消费端、提供端的调用链
	bootstrap.SetHandlers()
	//加载配置文件
	//microservice.yaml：服务的信息
	//chassis.yaml：注册中心、本地服务暴露的协议、地址
	if err := chassis.Init(); err != nil {
		openlogging.Error("Go chassis init failed, Mesher is not available: " + err.Error())
		panic(err)
	}
	//初始化出口调用链
	if err := bootstrap.InitEgressChain(); err != nil {
		openlogging.Error("egress chain int failed: %s", openlogging.WithTags(openlogging.Tags{
			"err": err.Error(),
		}))
		panic(err)
	}
	//初始化配置和组件
	if err := bootstrap.Start(); err != nil {
		openlogging.Error("Bootstrap failed: " + err.Error())
		panic(err)
	}
	openlogging.Info("server start complete", openlogging.WithTags(openlogging.Tags{
		"version": version.Ver().Version,
	}))
	//监听本地服务的健康状况
	if err := health.Run(); err != nil {
		openlogging.Error("Health manager start failed: " + err.Error())
		panic(err)
	}
	profile()
	//启动微服务，对外提供服务
	chassis.Run()
}
```

## CmdInit

解析命令行参数

proxy/cmd/cmd.go

```go
var Configs *ConfigFromCmd //全局变量，存放命令行参数
```

```go
func Init() error {
   Configs = &ConfigFromCmd{}
   return parseConfigFromCmd(os.Args)
}
```

proxy/cmd/cmd.go

```go
func parseConfigFromCmd(args []string) (err error) { //解析参数
   app := cli.NewApp()
   app.HideVersion = true
   app.Usage = "a service mesh that governance your service traffic."
   app.Flags = []cli.Flag{
      cli.StringFlag{
         Name:        "config",
         Usage:       "mesher config file, example: --config=mesher.yaml",
         Destination: &Configs.ConfigFile,
      },
      cli.StringFlag{
         Name:  "mode",
         Value: common.RoleSidecar,
         Usage: fmt.Sprintf("mesher role [ %s|%s|%s ]",
            common.RolePerHost, common.RoleSidecar, common.RoleEdge),
         Destination: &Configs.Role,
      },
      cli.StringFlag{
         Name:        "service-ports",
         EnvVar:      common.EnvServicePorts,
         Usage:       fmt.Sprintf("service protocol and port,examples: --service-ports=http:3000,grpc:8000"),
         Destination: &Configs.LocalServicePorts,
      },
   }
   app.Action = func(c *cli.Context) error {
      return nil
   }

   err = app.Run(args)
   return
}
```



## RegisterFramework

proxy/bootstrap/bootstrap.go:

```go
func RegisterFramework() {
   version := GetVersion()
   if framework := metadata.NewFramework(); cmd.Configs.Role == common.RoleSidecar {
      framework.SetName("Mesher")
      framework.SetVersion(version)
      framework.SetRegister("SIDECAR")
   } else {
      framework.SetName("Mesher")
      framework.SetVersion(version)
   }
}
```

## SetHandlers

设置消费端、提供端的调用链

proxy/bootstrap/bootstrap.go

```go
func SetHandlers() {
   consumerChain := strings.Join([]string{
      chassisHandler.Router,
      chassisHandler.RateLimiterConsumer,
      chassisHandler.BizkeeperConsumer,
      chassisHandler.Loadbalance,
      chassisHandler.Transport,
   }, ",") //消费端
   providerChain := strings.Join([]string{
      chassisHandler.RateLimiterProvider,
      chassisHandler.Transport,
   }, ",") //提供端
   consumerChainMap := map[string]string{
      common.ChainConsumerOutgoing: consumerChain,
   }
   providerChainMap := map[string]string{
      common.ChainProviderIncoming: providerChain,
      "default":                    chassisHandler.RateLimiterProvider,
   }
   chassis.SetDefaultConsumerChains(consumerChainMap)
   chassis.SetDefaultProviderChains(providerChainMap)
}
```

## InitEgressChain

初始化出口调用链

```go
func InitEgressChain() error {
   egresschain := strings.Join([]string{
      handler.RateLimiterConsumer,
      handler.BizkeeperConsumer,
      handler.Transport,
   }, ",")
   egressChainMap := map[string]string{
      common.ChainConsumerEgress: egresschain,
   }
   return handler.CreateChains(common.ConsumerEgress, egressChainMap)
}
```

## BootstrapStart

proxy/bootstrap/bootstrap.go

初始化配置和组件

```go
func Start() error {
   if err := config.InitProtocols(); err != nil {
      return err
   }
   //加载mesher.yaml、egress.yaml文件
   //创建全局的MesherConfig、EgressConfig
   if err := config.Init(); err != nil {
      return err
   }
   //创建DestinationResolver
   if err := resolver.Init(); err != nil {
      return err
   }
   if err := DecideMode(); err != nil {
      return err
   }
   metrics.Init()
   if err := v1.Init(); err != nil {
      log.Println("Error occurred in starting admin server", err)
   }
   if err := register.AdaptEndpoints(); err != nil {
      return err
   }
   if cmd.Configs.LocalServicePorts == "" {
      lager.Logger.Warnf("local service ports is missing, service can not be called by mesher")
   } else {
      lager.Logger.Infof("local service ports is [%v]", cmd.Configs.PortsMap)
   }
   err := egress.Init()
   if err != nil {
      return err
   }
   //初始化面板
   if err := control.Init(); err != nil {
      return err
   }
   return nil
}
```

### Config

加载mesher.yaml、egress.yaml文件,创建全局的MesherConfig、EgressConfig

proxy/config/config.go

```go
func Init() error {
   mesherConfig = &MesherConfig{}
   egressConfig = &EgressConfig{}
   p, err := GetConfigFilePath(ConfFile)
   if err != nil {
      return err
   }
   err = archaius.AddFile(p)
   if err != nil {
      return err
   }

   contents, err := GetConfigContents(ConfFile)//加载mesher.yaml
   if err != nil {
      return err
   }
   if err := yaml.Unmarshal([]byte(contents), mesherConfig); err != nil {
      return err
   }

   egressContents, err := GetConfigContents(EgressConfFile)//加载egress.yaml文件
   if err != nil {
      return err
   }
   if err := yaml.Unmarshal([]byte(egressContents), egressConfig); err != nil {
      return err
   }
   return nil
}
```

### Resolver

目标地址解析器，mesher会对本地服务进行代理，本地服务对外的访问都要通过mesher，mesher接收到本地服务请求后，会对请求进行解析

proxy/resolver/destination.go

```go
func Init() error {
   if config.GetConfig().Plugin != nil {
      for name, v := range config.GetConfig().Plugin.DestinationResolver {
         if v == "" {
            v = DefaultPlugin
         }
         f, ok := DestinationResolverPlugins[v]
         if !ok {
            return fmt.Errorf("unknown destination resolver [%s]", v)
         }
         drMap[name] = f() //调用函数，生成对象
      }
   }
   return nil
}
```

### Egress

proxy/pkg/egress/egress_config.go

```go
func Init() error {
   // 获取egress配置
   egressConfigFromFile := config.GetEgressConfig()
   //创建cseEgress	
   err := BuildEgress(GetEgressType(egressConfigFromFile.Egress))
   if err != nil {
      return err
   }
   op, err := getSpecifiedOptions()
   if err != nil {
      return fmt.Errorf("egress options error: %v", err)
   }
   err = DefaultEgress.Init(op)
   if err != nil {
      return err
   }
   openlogging.Info("Egress init success")
   return nil
}
```

proxy/pkg/egress/archaius/egress.go

```go
func (r *Egress) Init(op egress.Options) error {
   // the manager use dests to init, so must init after dests
   if err := initEgressManager(); err != nil {
      return err
   }
   return refresh()
}
```

### control

proxy/control/panel.go

```go
func Init() error {
   infra := config.GlobalDefinition.Panel.Infra
   if infra == "" || infra == "archaius" {
      return nil
   }

   f, ok := panelPlugin[infra]
   if !ok {
      return fmt.Errorf("do not support [%s] panel", infra)
   }

   DefaultPanelEgress = f(Options{
      Address: config.GlobalDefinition.Panel.Settings["address"],
   })
   return nil
}
```

## Health

监听本地服务的健康状态

proxy/health/health.go

```go
func Run() error {
   openlogging.Info("local health manager start")
   for _, v := range config.GetConfig().HealthCheck {
      lager.Logger.Debugf("check local health [%s],protocol [%s]", v.Port, v.Protocol)
      address, check, err := ParseConfig(v)
      if err != nil {
         lager.Logger.Warn("Health keeper can not check health")
         return err
      }
      //TODO make pluggable Deal
      if err := runCheckers(v, check, address, UpdateInstanceStatus); err != nil {
         return err
      }
   }
   return nil
}
```

# Http协议

proxy/protocol/http/http_server.go

```go
func (hs *httpServer) Start() error { //启动Http服务
   host, port, err := net.SplitHostPort(hs.opts.Address)
   if err != nil {
      return err
   }
   ip := net.ParseIP(host)
   if ip == nil {
      return fmt.Errorf("IP format error, input is [%s]", hs.opts.Address)
   }
   if ip.To4() == nil {
      return fmt.Errorf("only support ipv4, input is [%s]", hs.opts.Address)
   }

   switch runtime.Role {
   case common.RoleSidecar: //sidecar
      err = hs.startSidecar(host, port) //启动sidecar
   default:
      err = hs.startCommonProxy()
   }
   if err != nil {
      return err
   }
   return nil
}
```

## 启动sidecar

proxy/protocol/http/http_server.go

```go
func (hs *httpServer) startSidecar(host, port string) error {
   mesherTLSConfig, mesherSSLConfig, mesherErr := chassisTLS.GetTLSConfigByService(
      common.ComponentName, "", chassisCom.Provider)
   if mesherErr != nil {
      if !chassisTLS.IsSSLConfigNotExist(mesherErr) {
         return mesherErr
      }
   } else {
      sslTag := genTag(common.ComponentName, chassisCom.Provider)
      lager.Logger.Warnf("%s TLS mode, verify peer: %t, cipher plugin: %s.",
         sslTag, mesherSSLConfig.VerifyPeer, mesherSSLConfig.CipherPlugin)
   }
   //监听本机请求，本机当做消费者，向远端发起请求
   err := hs.listenAndServe("127.0.0.1"+":"+port, mesherTLSConfig, http.HandlerFunc(LocalRequestHandler))
   if err != nil {
      return err
   }
   resolver.SelfEndpoint = "127.0.0.1" + ":" + port

   switch host {
   case "0.0.0.0": //sidecar禁止监听0.0.0.0
      return errors.New("in sidecar mode, forbidden to listen on 0.0.0.0")
   case "127.0.0.1": //消费端会监听127.0.0.1
      lager.Logger.Warnf("Mesher listen on 127.0.0.1, it can only proxy for consumer. " +
         "for provider, mesher must listen on external ip.")
      return nil
   default:
      serverTLSConfig, serverSSLConfig, serverErr := chassisTLS.GetTLSConfigByService(
         chassisRuntime.ServiceName, chassisCom.ProtocolRest, chassisCom.Provider)
      if serverErr != nil {
         if !chassisTLS.IsSSLConfigNotExist(serverErr) {
            return serverErr
         }
      } else {
         sslTag := genTag(chassisRuntime.ServiceName, chassisCom.ProtocolRest, chassisCom.Provider)
         lager.Logger.Warnf("%s TLS mode, verify peer: %t, cipher plugin: %s.",
            sslTag, serverSSLConfig.VerifyPeer, serverSSLConfig.CipherPlugin)
      }
      //监听远端请求，本机当做提供者
      err = hs.listenAndServe(hs.opts.Address, serverTLSConfig, http.HandlerFunc(RemoteRequestHandler))
      if err != nil {
         return err
      }
   }
   return nil
}
```

### LocalRequestHandler

充当消费端，调用远程服务

proxy/protocol/http/sidecar.go

```go
func LocalRequestHandler(w http.ResponseWriter, r *http.Request) {
   prepareRequest(r)
   inv := consumerPreHandler(r)
   remoteIP := stringutil.SplitFirstSep(r.RemoteAddr, ":")

   var err error
   h := make(map[string]string)
   for k := range r.Header {
      h[k] = r.Header.Get(k)
   }
   //解析出目标地址，微服务名称
   destination, port, err := dr.Resolve(remoteIP, r.Host, r.URL.String(), h)
   if err != nil {
      handleErrorResponse(inv, w, http.StatusBadRequest, err)
      return
   }
   inv.MicroServiceName = destination
   if port != "" {
      h[XForwardedPort] = port
   }

   //transfer header into ctx
   inv.Ctx = context.WithValue(inv.Ctx, chassisCommon.ContextHeaderKey{}, h)

   var c *handler.Chain
  //根据服务名称获取egressRule
   ok, egressRule := egress.Match(inv.MicroServiceName)
   if ok {
      var targetPort int32 = 80
      for _, port := range egressRule.Ports {
         if strings.EqualFold(port.Protocol, common.HTTPProtocol) {
            targetPort = port.Port
            break
         }
      }
      inv.Endpoint = inv.MicroServiceName + ":" + strconv.Itoa(int(targetPort))
     	//获取调用链
      c, err = handler.GetChain(common.ConsumerEgress, common.ChainConsumerEgress)
      if err != nil {
         handleErrorResponse(inv, w, http.StatusBadGateway, err)
         openlogging.Error("Get chain failed" + err.Error())
         return
      }
   } else {
     	//获取调用链
      c, err = handler.GetChain(chassisCommon.Consumer, common.ChainConsumerOutgoing)
      if err != nil {
         handleErrorResponse(inv, w, http.StatusBadGateway, err)
         openlogging.Error("Get chain failed: " + err.Error())
         return
      }
   }
   defer func(begin time.Time) {
      timeTaken := time.Since(begin).Seconds()
      serviceLabelValues := map[string]string{metrics.LServiceName: inv.MicroServiceName, metrics.LApp: inv.RouteTags.AppID(), metrics.LVersion: inv.RouteTags.Version()}
      metrics.RecordLatency(serviceLabelValues, timeTaken)
   }(time.Now())
  
   var invRsp *invocation.Response
   c.Next(inv, func(ir *invocation.Response) error {
      //Send the request to the destination
      invRsp = ir
      if invRsp != nil {
         return invRsp.Err
      }
      return nil
   })
   resp, err := handleRequest(w, inv, invRsp)
   if err != nil {
      openlogging.Error("handle request failed: " + err.Error())
      return
   }
   RecordStatus(inv, resp.StatusCode)
}
```

### RemoteRequestHandler

充当服务提供者，接收外部请求，调用本机代理的服务

```go
func RemoteRequestHandler(w http.ResponseWriter, r *http.Request) {
	//准备请求
	prepareRequest(r)
	//创建Invocation
	inv := providerPreHandler(r)
	if inv.SourceMicroService == "" {
		source := stringutil.SplitFirstSep(r.RemoteAddr, ":")
		//解析外部请求的服务名
		si := sr.Resolve(source)
		if si != nil {
			inv.SourceMicroService = si.Name
		}
	}
	//获取请求头
	h := make(map[string]string)
	for k := range r.Header {
		h[k] = r.Header.Get(k)
	}
	//将请求头填充到ctx
	inv.Ctx = context.WithValue(inv.Ctx, chassisCommon.ContextHeaderKey{}, h)
	//获取调用链
	c, err := handler.GetChain(chassisCommon.Provider, common.ChainProviderIncoming)
	if err != nil {
		handleErrorResponse(inv, w, http.StatusBadGateway, err)
		lager.Logger.Error("Get chain failed: " + err.Error())
		return
	}
	if err = util.SetLocalServiceAddress(inv, r.Header.Get("X-Forwarded-Port")); err != nil {
		handleErrorResponse(inv, w, http.StatusBadGateway,
			err)
	}
	if r.Header.Get(XForwardedHost) == "" {
		r.Header.Set(XForwardedHost, r.Host)
	}
	var invRsp *invocation.Response
	//调用本机服务
	c.Next(inv, func(ir *invocation.Response) error {
		//Send the request to the destination
		invRsp = ir
		if invRsp != nil {
			return invRsp.Err
		}
		return nil
	})
	//处理响应结果
	if _, err = handleRequest(w, inv, invRsp); err != nil {
		lager.Logger.Error("Handle request failed: " + err.Error())
	}
}
```

