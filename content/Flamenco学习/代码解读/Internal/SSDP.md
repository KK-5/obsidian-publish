flamenco使用ssdp协议来连接manager和worker，关于SSDP（简单服务发现协议）：[https://zh.wikipedia.org/wiki/%E7%AE%80%E5%8D%95%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0%E5%8D%8F%E8%AE%AE](https://zh.wikipedia.org/wiki/%E7%AE%80%E5%8D%95%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0%E5%8D%8F%E8%AE%AE)

ssdp协议规定client接入网络时可以向一个server发送信息，server会根据信息判断自己是否有client请求的服务，如果有就响应它。同理server接入网络时会发送一次广播，client根据自己的策略处理接受到的信息。

floamenco使用 github.com/fromkeith/gossdp 包的ssdp协议功能。

## Client

```
type Client struct {
	ssdp       *gossdp.ClientSsdp
	log        *zerolog.Logger
	wrappedLog *ssdpLogger

	mutex    *sync.Mutex
	urls     []string        // Preserves order
	seenURLs map[string]bool // Removes duplicates
}

func NewClient(logger zerolog.Logger) (*Client, error) {
	wrap := wrappedLogger(&logger)
	client := Client{
		log:        &logger,
		wrappedLog: wrap,

		mutex:    new(sync.Mutex),
		urls:     make([]string, 0),
		seenURLs: make(map[string]bool),
	}

	ssdp, err := gossdp.NewSsdpClientWithLogger(&client, wrap)
	if err != nil {
		return nil, fmt.Errorf("create UPnP/SSDP client: %w", err)
	}

	client.ssdp = ssdp
	return &client, nil
}
```

Client的核心是*gossdp.ClientSsdp指针，提供ssdp client的功能，urls保存它可以看到的server地址，seenURLs快速判断此url是否在它的保存的urls中。

### Run

```
func (c *Client) Run(ctx context.Context) ([]string, error) {
    defer c.stopCleanly()

    log.Debug().Msg("waiting for UPnP/SSDP answer")
    go c.ssdp.Start()

    var waitTime time.Duration
    for {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(waitTime):
            if err := c.ssdp.ListenFor(FlamencoServiceType); err != nil {
                return nil, fmt.Errorf("unable to find Manager: %w", err)
            }
            waitTime = 1 * time.Second

            urls := c.receivedURLs()
            if len(urls) > 0 {
                return urls, nil
            }
        }
    }
}
```

启动ssdp客户端并等待应答，如果超时自动关闭client。

## Server

```
// Server advertises services via UPnP/SSDP.
type Server struct {
	ssdp       *gossdp.Ssdp
	log        *zerolog.Logger
	wrappedLog *ssdpLogger
}

func NewServer(logger zerolog.Logger) (*Server, error) {
	wrap := wrappedLogger(&logger)
	ssdp, err := gossdp.NewSsdpWithLogger(nil, wrap)
	if err != nil {
		return nil, err
	}
	return &Server{ssdp, &logger, wrap}, nil
}
```

和Client一样，Server核心是一个gossdp.Ssdp指针。

### AddAdvertisement

```
// AddAdvertisement adds a service advertisement for Flamenco Manager.
// Must be called before calling Run().
func (s *Server) AddAdvertisement(serviceLocation string) {
	// Define the service we want to advertise
	serverDef := gossdp.AdvertisableServer{
		ServiceType: FlamencoServiceType,  // "urn:flamenco:manager:0"
		DeviceUuid:  FlamencoUUID,    // "aa80bc5f-d0af-46b8-8630-23bd7e80ec4d"
		Location:    serviceLocation,  
		MaxAge:      3600, // Number of seconds this advertisement is valid for.
	}
	s.ssdp.AdvertiseServer(serverDef)
	s.log.Debug().Str("location", serviceLocation).Msg("UPnP/SSDP location registered")
}

// AddAdvertisementURLs constructs a service location from the given URLs, and
// adds the advertisement for it.
func (s *Server) AddAdvertisementURLs(baseURLs []url.URL) {
	for _, url := range baseURLs {
		url.Path = path.Join(url.Path, serviceDescriptionPath)
		s.AddAdvertisement(url.String())
	}
}
```

添加广播地址，先添加之后才能启动server。

### Run

```
// Run starts the advertisement, and blocks until the context is closed.
func (s *Server) Run(ctx context.Context) {
	s.log.Info().Msg("UPnP/SSDP advertisement starting")

	isStopping := false

	go func() {
		// There is a bug in the SSDP library, where closing the server can cause a panic.
		defer func() {
			if isStopping {
				// Only capture a panic when we expect one.
				value := recover()
				s.log.Debug().Interface("value", value).Msg("recovered from panic in SSDP library")
			}
		}()

		s.ssdp.Start()
	}()

	<-ctx.Done()

	s.log.Debug().Msg("UPnP/SSDP advertisement stopping")

	// Sneakily disable warnings when shutting down, otherwise the read operation
	// from the UDP socket will cause a warning.
	tempLog := s.log.Level(zerolog.ErrorLevel)
	s.wrappedLog.zlog = &tempLog
	isStopping = true
	s.ssdp.Stop()
	s.wrappedLog.zlog = s.log

	s.log.Info().Msg("UPnP/SSDP advertisement stopped")
}
```

启动server服务。