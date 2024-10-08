这是[[CommandExecutor]]中注册的一种可执行的命令，也是flamenco最核心的渲染功能执行的命令。在CommandExecutor中，它的registry初始化是这样的：

```
    ce.registry = map[string]commandCallable{
        // misc
        "echo":  ce.cmdEcho,
        "sleep": ce.cmdSleep,
        "exec":  ce.cmdExec,
        // blender
        "blender-render": ce.cmdBlenderRender,
        // ffmpeg
        "frames-to-video": ce.cmdFramesToVideo,
        // file-management
        "move-directory": ce.cmdMoveDirectory,
        "copy-file":      ce.cmdCopyFile,
    }
```

在这里可以看到blender-render对应的执行命令的函数就是cmdBlenderRender。

cmdBlenderRender函数代码：

```
// cmdBlender executes the "blender-render" command.
func (ce *CommandExecutor) cmdBlenderRender(ctx context.Context, logger zerolog.Logger, taskID string, cmd api.Command) error {
    cmdCtx, cmdCtxCancel := context.WithCancel(ctx)
    defer cmdCtxCancel() // Ensure the subprocess exits whenever this function returns.

    execCmd, err := ce.cmdBlenderRenderCommand(cmdCtx, logger, taskID, cmd)
    if err != nil {
        return err
    }
    logChunker := NewLogChunker(taskID, ce.listener, ce.timeService)
    lineChannel := make(chan string)
    // Process the output of Blender in its own goroutine.
    wg := sync.WaitGroup{}
    wg.Add(1)
    go func() {
        defer wg.Done()
        for line := range lineChannel {
            ce.processLineBlender(ctx, logger, taskID, line)
        }
    }()

    // Run the subprocess.
    subprocessErr := ce.cli.RunWithTextOutput(ctx,
        logger,
        execCmd,
        logChunker,
        lineChannel,
    )

    // Wait for the processing to stop.
    close(lineChannel)
    wg.Wait()
    ...
    return nil
}
```

从上面的函数中可以看出当执行blender渲染的命令时，先从cmdBlenderRenderCommand这个函数初始化出一个execCmd，然后在下面开启了一个goroutine，在其中遍历了lineChannel中的每一行line，然后使用processLineBlender对其进行处理，再下面，RunWithTextOutput开启一个子进程来执行命令，并将命令执行产生的cmd输出放到lineChannel中。

看一下processLineBlender如何处理日志的，

```
var regexpFileSaved = regexp.MustCompile("Saved: '(.*)'")

func (ce *CommandExecutor) processLineBlender(ctx context.Context, logger zerolog.Logger, taskID string, line string) {
    // TODO: check for "Warning: Unable to open" and other indicators of missing
    // files. Flamenco v2 updated the task.Activity field for such situations.

    match := regexpFileSaved.FindStringSubmatch(line)
    if len(match) < 2 {
        return
    }
    filename := match[1]

    logger = logger.With().Str("outputFile", filename).Logger()
    logger.Info().Msg("output produced")

    err := ce.listener.OutputProduced(ctx, taskID, filename)
    if err != nil {
        logger.Warn().Err(err).Msg("error submitting produced output to listener")
    }
}
```

它从命令执行的输出中找到带有‘Saved:’的那行，然后从中提取出blender渲染产生的文件，再使用OutputProduced上传至manager。在blender中，渲染图片时会有这样一条日志：
![[Pasted image 20240701162039.png]]

这里就是通过这条日志来找到渲染输出文件的。

其他类型的命令也是通过类似的方法执行。