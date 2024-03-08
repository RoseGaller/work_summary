# nsqlookupd

nsqlookupd/main.go

```go
var (
   showVersion = flag.Bool("version", false, "print version string")
   tcpAddress  = flag.String("tcp-address", "0.0.0.0:4160", "<addr>:<port> to listen on for TCP clients")
   httpAddress = flag.String("http-address", "0.0.0.0:4161", "<addr>:<port> to listen on for HTTP clients")
   debugMode   = flag.Bool("debug", false, "enable debug mode")
)
var protocols = map[int32]nsq.Protocol{}  //全局变量
```

nsqlookupd/lookup_protocol_v1.go

```go
func init() {
   // BigEndian client byte sequence "  V1"
   var magicInt int32
   buf := bytes.NewBuffer([]byte(nsq.MagicV1))
   binary.Read(buf, binary.BigEndian, &magicInt)
   protocols[magicInt] = &LookupProtocolV1{} //注册Protocol，处理tcp
}
```

```go
func main() {
   flag.Parse()

   if *showVersion {
      fmt.Printf("nsqlookupd v%s\n", util.BINARY_VERSION)
      return
   }

   signalChan := make(chan os.Signal, 1)
   exitChan := make(chan int)
   go func() {
      <-signalChan
      exitChan <- 1
   }()
   signal.Notify(signalChan, os.Interrupt)

   tcpAddr, err := net.ResolveTCPAddr("tcp", *tcpAddress)
   if err != nil {
      log.Fatal(err)
   }

   httpAddr, err := net.ResolveTCPAddr("tcp", *httpAddress)
   if err != nil {
      log.Fatal(err)
   }

   log.Printf("nsqlookupd v%s", util.BINARY_VERSION)

   lookupd = NewNSQLookupd()
   lookupd.tcpAddr = tcpAddr
   lookupd.httpAddr = httpAddr
   lookupd.Main()
   <-exitChan
   lookupd.Exit()
}
```

## NewNSQLookupd

nsqlookupd/nsqlookupd.go

```go
func NewNSQLookupd() *NSQLookupd {
   return &NSQLookupd{
      DB: NewRegistrationDB(),
   }
}
```

nsqlookupd/registration_db.go

```go
func NewRegistrationDB() *RegistrationDB {
   return &RegistrationDB{
      registrationMap: make(map[Registration]Producers),
   }
}
```

## Main

nsqlookupd/nsqlookupd.go

```go
func (l *NSQLookupd) Main() {
   tcpListener, err := net.Listen("tcp", l.tcpAddr.String())
   if err != nil {
      log.Fatalf("FATAL: listen (%s) failed - %s", l.tcpAddr, err.Error())
   }
   l.tcpListener = tcpListener
   l.waitGroup.Wrap(func() { util.TcpServer(tcpListener, &TcpProtocol{protocols: protocols}) })

   httpListener, err := net.Listen("tcp", l.httpAddr.String())
   if err != nil {
      log.Fatalf("FATAL: listen (%s) failed - %s", l.httpAddr, err.Error())
   }
   l.httpListener = httpListener
   l.waitGroup.Wrap(func() { httpServer(httpListener) })

}
```

### TCP

util/tcp_server.go

```go
func TcpServer(listener net.Listener, handler TcpHandler) {
   log.Printf("TCP: listening on %s", listener.Addr().String())

   for {
      clientConn, err := listener.Accept()
      if err != nil {
         if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
            log.Printf("NOTICE: temporary Accept() failure - %s", err.Error())
            runtime.Gosched()
            continue
         }
         // theres no direct way to detect this error because it is not exposed
         if !strings.Contains(err.Error(), "use of closed network connection") {
            log.Printf("ERROR: listener.Accept() - %s", err.Error())
         }
         break
      }
      go handler.Handle(clientConn)
   }

   log.Printf("TCP: closing %s", listener.Addr().String())
}
```

nsqlookupd/tcp.go

```go
func (p *TcpProtocol) Handle(clientConn net.Conn) {
   log.Printf("TCP: new client(%s)", clientConn.RemoteAddr())

   protocolMagic, err := nsq.ReadMagic(clientConn)
   if err != nil {
      log.Printf("ERROR: failed to read protocol version - %s", err.Error())
      return
   }

   log.Printf("CLIENT(%s): desired protocol %d", clientConn.RemoteAddr(), protocolMagic)

   prot, ok := p.protocols[protocolMagic] //默认LookupProtocolV1
   if !ok {
      nsq.SendResponse(clientConn, []byte("E_BAD_PROTOCOL"))
      log.Printf("ERROR: client(%s) bad protocol version %d", clientConn.RemoteAddr(), protocolMagic)
      return
   }

   err = prot.IOLoop(clientConn)
   if err != nil {
      log.Printf("ERROR: client(%s) - %s", clientConn.RemoteAddr(), err.Error())
      return
   }
}
```

nsqlookupd/lookup_protocol_v1.go

```go
func (p *LookupProtocolV1) IOLoop(conn net.Conn) error {
   var err error
   var line string

   client := NewClientV1(conn)
   client.State = nsq.StateInit
   err = nil
   reader := bufio.NewReader(client)
   for {
      line, err = reader.ReadString('\n')
      if err != nil {
         break
      }

      line = strings.TrimSpace(line)
      params := strings.Split(line, " ")

      response, err := p.Exec(client, reader, params)
      if err != nil {
         log.Printf("ERROR: CLIENT(%s) - %s", client, err.(*nsq.ClientErr).Description())
         _, err = nsq.SendResponse(client, []byte(err.Error()))
         if err != nil {
            break
         }
         continue
      }

      if response != nil {
         _, err = nsq.SendResponse(client, response)
         if err != nil {
            break
         }
      }
   }

   log.Printf("CLIENT(%s): closing", client)
   if client.Producer != nil {
      lookupd.DB.Remove(Registration{"client", "", ""}, client.Producer)
      registrations := lookupd.DB.LookupRegistrations(client.Producer)
      for _, r := range registrations {
         lookupd.DB.Remove(*r, client.Producer)
      }
   }
   return err
}
```

```go
func (p *LookupProtocolV1) Exec(client *ClientV1, reader *bufio.Reader, params []string) ([]byte, error) {
   switch params[0] {
   case "PING":
      return p.PING(client, params)
   case "IDENTIFY":
      return p.IDENTIFY(client, reader, params[1:])
   case "REGISTER":
      return p.REGISTER(client, reader, params[1:])
   case "UNREGISTER":
      return p.UNREGISTER(client, reader, params[1:])
   case "ANNOUNCE":
      return p.ANNOUNCE_OLD(client, reader, params[1:])
   }
   return nil, nsq.NewClientErr("E_INVALID", fmt.Sprintf("invalid command %s", params[0]))
}
```

### HTTP

