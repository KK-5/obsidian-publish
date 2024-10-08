在flamenco中，worker和manager使用ssdp进行连接，所以本质上manager和worker是一个C/S架构，即manager作为server提供各种服务，worker作为client向server请求服务。

manager提供的服务通过各类接口来实现，同样，worker也通过各类接口向manager请求数据，这些接口都是通过OpenAPI生成的。

这里的client指的是worker中的对象，它复制与manager的交互，最重要的是从manager处获取任务信息。

首先看worker的初始化函数：

```
// NewWorker constructs and returns a new Worker.
func NewWorker(
    flamenco FlamencoClient,
    taskRunner TaskRunner,
) *Worker {

    worker := &Worker{
        doneChan: make(chan struct{}),
        doneWg:   new(sync.WaitGroup),
        shutdown: make(chan struct{}),

        client: flamenco,

        state:         api.WorkerStatusStarting,
        stateStarters: make(map[api.WorkerStatus]StateStarter),
        stateMutex:    new(sync.Mutex),

        taskRunner: taskRunner,
    }
    worker.setupStateMachine()
    return worker
}
```

它在初始化时首先就需要一个FlamencoClient类型的对象，它的定义如下：

```
// FlamencoClient is a wrapper for api.ClientWithResponsesInterface so that locally mocks can be created.
type FlamencoClient interface {
    api.ClientWithResponsesInterface
}
```

在它的描述中提到FlamencoClient是api.ClientWithResponsesInterface的包装器，以便于生成mock，所以client本质就是一个api.ClientWithResponsesInterface，这是一个完全由OpenAPI生成的类，

```
// ClientWithResponsesInterface is the interface specification for the client with responses above.
type ClientWithResponsesInterface interface {
    // GetConfiguration request
    GetConfigurationWithResponse(ctx context.Context, reqEditors ...RequestEditorFn) (*GetConfigurationResponse, error)

    // FindBlenderExePath request
    FindBlenderExePathWithResponse(ctx context.Context, reqEditors ...RequestEditorFn) (*FindBlenderExePathResponse, error)

    // CheckBlenderExePath request with any body
    CheckBlenderExePathWithBodyWithResponse(ctx context.Context, contentType string, body io.Reader, reqEditors ...RequestEditorFn) (*CheckBlenderExePathResponse, error)

    CheckBlenderExePathWithResponse(ctx context.Context, body CheckBlenderExePathJSONRequestBody, reqEditors ...RequestEditorFn) (*CheckBlenderExePathResponse, error)
    
    ...
}
```

ClientWithResponsesInterface是一个可以获得请求Response信息的接口，其中的每一个请求都对应manager提供的一个服务，上面提到它是一个特殊的版本，那肯定还有一个普通的版本，

```
// The interface specification for the client above.
type ClientInterface interface {
    // GetConfiguration request
    GetConfiguration(ctx context.Context, reqEditors ...RequestEditorFn) (*http.Response, error)

    // FindBlenderExePath request
    FindBlenderExePath(ctx context.Context, reqEditors ...RequestEditorFn) (*http.Response, error)

    // CheckBlenderExePath request with any body
    CheckBlenderExePathWithBody(ctx context.Context, contentType string, body io.Reader, reqEditors ...RequestEditorFn) (*http.Response, error)

    CheckBlenderExePath(ctx context.Context, body CheckBlenderExePathJSONRequestBody, reqEditors ...RequestEditorFn) (*http.Response, error)
    ...
}
```

这个版本更容易看出，每一个接口都是请求的manger上的一个api。

关于client是如何创建的，

```
// NewClientWithResponses creates a new ClientWithResponses, which wraps
// Client with return type handling
func NewClientWithResponses(server string, opts ...ClientOption) (*ClientWithResponses, error) {
    client, err := NewClient(server, opts...)
    if err != nil {
        return nil, err
    }
    return &ClientWithResponses{client}, nil
}
```

这些都是OpenAPI自动生成的代码。

总结下来，worker中与manager交互的功能都由client实现，client本质是发送http请求来调用manager提供的API，也就是说，如果可以自己完成worker上的各种任务执行机制，用户是可以自己编写自己的worker程序而不用使用flamenco提供的worker的。