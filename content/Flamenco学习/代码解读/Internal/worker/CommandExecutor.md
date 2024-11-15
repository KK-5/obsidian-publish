顾名思义，这个类用于执行worker上需要执行的各种命令，是worker中最核心的功能。先看它的定义：

```go
type CommandExecutor struct {
	cli         CommandLineRunner
	listener    CommandListener
	timeService TimeService

	// registry maps a command name to a function that runs that command.
	registry map[string]commandCallable
}
```

CommandExecutor中有四个变量：
cli：用于在本计算机上执行cli命令。
listener：用于将命令执行的结果和日志上传至manager，以便于查看任务的执行情况。
timeService：时间服务。
registry：保存每种任务和它们需要执行的函数的映射，比如blender-render类型的任务就需要执行计算机上的blender程序。commandCallable的定义为：
```go
type commandCallable func(ctx context.Context, logger zerolog.Logger, taskID string, cmd api.Command) error
```

初始化：

```go
func NewCommandExecutor(cli CommandLineRunner, listener CommandListener, timeService TimeService) *CommandExecutor {
	ce := &CommandExecutor{
		cli:         cli,
		listener:    listener,
		timeService: timeService,
	}
	// Registry of supported commands. Having this as a map (instead of a big
	// switch statement) makes it possible to do things like reporting the list of
	// supported commands.
	ce.registry = map[string]commandCallable{
		// misc
		"echo":  ce.cmdEcho,
		"sleep": ce.cmdSleep,
		"exec":  ce.cmdExec,
		// blender
		"blender-render": ce.cmdBlenderRender,
		// ffmpeg
		"frames-to-video": ce.cmdFramesToVideo,
		// file-management
		"move-directory": ce.cmdMoveDirectory,
		"copy-file":      ce.cmdCopyFile,
	}
	return ce
}
```

CommandExecutor的初始化参数就是前3个成员变量，将成员变量都赋值后，再将registry初始化，可以看到每种类型的任务都有它们对应的commandCallable，比如echo对应ce.cmdEcho，这就是worker支持的所有任务类型。

## CommandLineRunner成员
```go
// CommandLineRunner is an interface around exec.CommandContext().
//
//go:generate go run github.com/golang/mock/mockgen -destination mocks/cli_runner.gen.go -package mocks projects.blender.org/studio/flamenco/internal/worker CommandLineRunner
type CommandLineRunner interface {
	CommandContext(ctx context.Context, name string, arg ...string) *exec.Cmd
	RunWithTextOutput(
		ctx context.Context,
		logger zerolog.Logger,
		execCmd *exec.Cmd,
		logChunker cli_runner.LogChunker,
		lineChannel chan<- string,
	) error
}
```

注释已经描述了，CommandLineRunner是一个包裹exec.CommandContext的接口，exec.CommandContext就是Go语言中执行CMD命令的工具。这个接口有两个函数，CommandContext用来获取exec.Cmd，RunWithTextOutput用来执行CMD命令。
### CLIRunner
实现CommandLineRunner这个接口的类叫做CLIRunner，CLIRunner中没有任何成员变量，单纯地实现了接口中的两个函数。
```go
type CLIRunner struct {
}
```
### CommandContext函数
```go
func (cli *CLIRunner) CommandContext(ctx context.Context, name string, arg ...string) *exec.Cmd {
	return exec.CommandContext(ctx, name, arg...)
}
```
非常简单，创建并返回一个exec.Cmd指针。
### RunWithTextOutput函数
```go
// RunWithTextOutput runs a command and sends its output line-by-line to the
// lineChannel. Stdout and stderr are combined.
// Before returning. RunWithTextOutput() waits for the subprocess, to ensure it
// doesn't become defunct.
//
// Note that all output read from the command is logged via `logChunker` as
// well, so the receiving end of the `lineChannel` does not have to do this.
func (cli *CLIRunner) RunWithTextOutput(
	ctx context.Context,
	logger zerolog.Logger,
	execCmd *exec.Cmd,
	logChunker LogChunker,
	lineChannel chan<- string,
) error {
	outPipe, err := execCmd.StdoutPipe()
	if err != nil {
		return err
	}
	execCmd.Stderr = execCmd.Stdout // Redirect stderr to stdout.
    ...
	if err := execCmd.Start(); err != nil {
		logger.Error().Err(err).Msg("error starting CLI execution")
		return err
	}

	subprocPID := execCmd.Process.Pid
	logger = logger.With().Int("pid", subprocPID).Logger()
    ...
}
```
这里只看它的核心代码部分，除了记录Cmd执行的日志以外，他唯一的功能就是execCmd.Start()开始执行Cmd命令。
## CommandListener成员
```go
// CommandListener sends the result of commands (log, output files) to the Manager.
type CommandListener interface {
    // LogProduced sends any logging to whatever service for storing logging.
    // logLines are concatenated.
    LogProduced(ctx context.Context, taskID string, logLines ...string) error
    // OutputProduced tells the Manager there has been some output (most commonly a rendered frame or video).
    OutputProduced(ctx context.Context, taskID string, outputLocation string) error
}
```
CommandListener也是一个接口，用于将worker的任务执行日志和任务结果提交给manager，用来观测各个任务的详细执行情况。
LogProduced：用于上传日志到manager。
OutputProduced：用于上传任务输出（一般是渲染图片）到manager。
### Listener
Listener是实现CommandListener接口的类。它的定义如下：
```go
// Listener listens to the result of task and command execution, and sends it to the Manager.
type Listener struct {
	client         FlamencoClient
	buffer         UpstreamBuffer
	outputUploader *OutputUploader
}
// NewListener creates a new Listener that will send updates to the API client.
func NewListener(client FlamencoClient, buffer UpstreamBuffer) *Listener {
	l := &Listener{
		client:         client,
		buffer:         buffer,
		outputUploader: NewOutputUploader(client),
	}
	return l
}
```
其中有3个成员变量，client用于与manager通信，buffer用来保存需要上传的信息，outputUploader实现上传操作，在它的初始化中，outputUploader也是需要client来进行初始化的，outputUploader就是Listener的核心。
#### LogProduced
```go
func (l *Listener) sendTaskUpdate(ctx context.Context, taskID string, update api.TaskUpdateJSONRequestBody) error {
	if ctx.Err() != nil {
		return ctx.Err()
	}
	return l.buffer.SendTaskUpdate(ctx, taskID, update)
}

// LogProduced sends any logging to whatever service for storing logging.
func (l *Listener) LogProduced(ctx context.Context, taskID string, logLines ...string) error {
    return l.sendTaskUpdate(ctx, taskID, api.TaskUpdateJSONRequestBody{
        Log: ptr(strings.Join(logLines, "\n")),
    })
}
```

LogProduced将最新的日志按行送到manager，其中调用了buffer.SendTaskUpdate方法，这是UpstreamBuffer中的一个方法。
日志传输的流程暂略。
#### OutputProduced
```go
// OutputProduced tells the Manager there has been some output (most commonly a rendered frame or video).
func (l *Listener) OutputProduced(ctx context.Context, taskID string, outputLocation string) error {
	l.outputUploader.OutputProduced(taskID, outputLocation)
	return nil
}
```
由outputUploader执行文件传输操作。

##### OutputUploader
这是执行真正文件传输操作的类，
```go
// OutputUploader sends (downscaled versions of) rendered images to Flamenco
// Manager. Only one image is sent at a time. A queue of a single image is kept,
// where newly queued images replace older ones.
type OutputUploader struct {
	client FlamencoClient
	queue  *last_in_one_out_queue.LastInOneOutQueue[TaskOutput]
}

type TaskOutput struct {
	TaskID   string
	Filename string
}

func NewOutputUploader(client FlamencoClient) *OutputUploader {
	return &OutputUploader{
		client: client,
		queue:  last_in_one_out_queue.New[TaskOutput](),
	}
}
```
可以看到，其中有一个FlamencoClient和一个queue，queue中保存了每一个任务的信息，信息由TaskID和Filename组成。
OutputUploader中的OutputProduced函数如下：
```go
// OutputProduced enqueues the given filename for processing.
func (ou *OutputUploader) OutputProduced(taskID, filename string) {
    // TODO: Before enqueueing (and thus overwriting any previously queued item),
    // check that this file can actually be handled by the Last Rendered system of
    // Flamenco. It would be a shame if a perfectly-good JPEG file is kicked off
    // the queue by an EXR file we can't handle.
    item := TaskOutput{taskID, filename}
    ou.queue.Enqueue(item)
}
```
它并未做任务传输操作，而是将需要传输的文件加入队列。
OutputUploader中还有一个Run函数，它是这样的：
```go
func (ou *OutputUploader) Run(ctx context.Context) {
    log.Info().Msg("output uploader: running")
    defer log.Info().Msg("output uploader: shutting down")

    wg := sync.WaitGroup{}
    wg.Add(1)
    go func() {
        defer wg.Done()
        ou.queue.Run(ctx)
    }()

runLoop:
    for {
        select {
        case <-ctx.Done():
            break runLoop
        case item := <-ou.queue.Item():
            ou.process(ctx, item)
        }
    }

    wg.Wait()
}
```
由此可以看出，worker的文件传输是异步的，它只要将需要传输的任务加入队列中，另一个goroutine会调用process函数来对每个任务进行处理，
process函数：
```go
// process loads the given image, converts it to JPEG, and uploads it to
// Flamenco Manager.
func (ou *OutputUploader) process(ctx context.Context, item TaskOutput) {
    ...
    jpegBytes := loadAsJPEG(item.Filename)
    if len(jpegBytes) == 0 {
        return // loadAsJPEG() already logged the error.
    }

    // Upload to Manager.
    jpegReader := bytes.NewReader(jpegBytes)
    resp, err := ou.client.TaskOutputProducedWithBodyWithResponse(
        ctx, item.TaskID, "image/jpeg", jpegReader)
    if err != nil {
        logger.Error().Err(err).Msg("output uploader: unable to send image to Manager")
        return
    }
    ...
}
```
将需要上传的图片解析为JPEG格式，然后上传到manager。
## CommandExecutor的其他成员函数
### Run
```go
func (ce *CommandExecutor) Run(ctx context.Context, taskID string, cmd api.Command) error {
    logger := log.With().Str("task", string(taskID)).Str("command", cmd.Name).Logger()
    logger.Info().Interface("parameters", cmd.Parameters).Msg("running Flamenco command")

    runner, ok := ce.registry[cmd.Name]
    if !ok {
        return fmt.Errorf("unknown command: %q", cmd.Name)
    }

    return runner(ctx, logger, taskID, cmd)
}
```
Run函数是CommandExecutor执行cmd命令的函数，传入一个cmd，根据cmd.Name在registry中找到一个commandCallable类型的函数，这里叫做runner，然后调用这个函数。registry中保存了每个cmd.Name和需要执行的函数的映射，在CommandExecutor构建时就初始化了，比如"blender-render"对应cmdBlenderRender，cmdBlenderRender中就会调用CLIRunner的RunWithTextOutput来执行cmd命令。
## 总结
CommandExecutor是worker中用来执行cmd命令的函数，在flamenco中，manager中的每一项任务最终都具体到一个cmd命令，可能是简单的echo命令，可能是文件管理命令，最常用的是blender xxxx的后台渲染命令。CommandExecutor提供了命令执行，日志上传，命令执行结果上传的功能，不过它可以执行的命令是有限制的，在初始化时所有可执行的命令类型都被记录在registry中。