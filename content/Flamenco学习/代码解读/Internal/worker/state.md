每个worker都有不同的state，用来管理worker在不同状态下的行为，比如一个worker在sleep状态，那么他不会接取manager上的任何任务，直到它的状态变为awake。

worker使用状态模式来管理自己的状态，也就是说，它的每个状态都是一个单独的模块，在其中定义了该状态下的行为。

下面以awake状态为例，看一下在工作状态下worker如何运行。

当worker切换为awake状态时，执行下面这个函数：

```
func (w *Worker) gotoStateAwake(ctx context.Context) {
    w.stateMutex.Lock()
    w.state = api.WorkerStatusAwake
    w.stateMutex.Unlock()

    w.doneWg.Add(2)
    w.ackStateChange(ctx, w.state)

    go w.runStateAwake(ctx)
}
```

第一步，状态切换到api.WorkerStatusAwake，这里使用mutex保证线程安全。

第二步，异步执行两个goroutine，ackStateChange是为了通知manager自已的状态变更；runStateAwake就是awake状态下worker应该执行的任务。

### runStateAwake

```
// runStateAwake fetches a task and executes it, in an endless loop.
func (w *Worker) runStateAwake(ctx context.Context) {
    ...
    for {
        task := w.fetchTask(ctx)
        if task == nil {
            return
        }

        // The task runner's listener will be responsible for sending results back
        // to the Manager. This code only needs to fetch a task and run it.
        err := w.runTask(ctx, *task)
        if err != nil {
           ...
        }

        // Do some rate limiting. This is mostly useful while developing.
        select {
        case <-ctx.Done():
            return
        case <-time.After(durationTaskComplete):
        }
    }
}
```

在这个函数中，worker一直fetchTask获取manager上还未执行的任务，然后runTask执行这个任务并将结果传回manager。

### fetchTask

```
// fetchTasks periodically tries to fetch a task from the Manager, returning it when obtained.
// Returns nil when a task could not be obtained and the period loop was cancelled.
func (w *Worker) fetchTask(ctx context.Context) *api.AssignedTask {
    ...
    var wait time.Duration
    for {
        select {
        case <-ctx.Done():
            logger.Debug().Msg("task fetching interrupted by context cancellation")
            return nil
        case <-w.doneChan:
            logger.Debug().Msg("task fetching interrupted by shutdown")
            return nil
        case <-time.After(wait):
        }

        logger.Debug().Msg("fetching tasks")
        resp, err := w.client.ScheduleTaskWithResponse(ctx)
        if err != nil {
            log.Error().Err(err).Msg("error obtaining task")
            wait = durationFetchFailed
            continue
        }
        switch {
        case resp.JSON200 != nil:
            log.Info().
                Interface("task", resp.JSON200).
                Msg("obtained task")
            return resp.JSON200
        case ...
            ...
        }
    }
}
```

fetchTask也是一个循环，每隔一段时间取获取一次任务（当前间隔时间是0），通过ScheduleTaskWithResponse获取到任务之后，将resp返回。resp是一个ScheduleTaskResponse类型的数据，它的定义如下：

```
type ScheduleTaskResponse struct {
    Body         []byte
    HTTPResponse *http.Response
    JSON200      *AssignedTask
    JSON403      *SecurityError
    JSON423      *WorkerStateChange
}
```

所以上面返回的resp.JSON200就是AssignedTask指针。也就是这样的结构体：

```
// AssignedTask is a task as it is received by the Worker.
type AssignedTask struct {
    Commands    []Command  `json:"commands"`
    Job         string     `json:"job"`
    JobPriority int        `json:"job_priority"`
    JobType     string     `json:"job_type"`
    Name        string     `json:"name"`
    Priority    int        `json:"priority"`
    Status      TaskStatus `json:"status"`
    TaskType    string     `json:"task_type"`
    Uuid        string     `json:"uuid"`
}
```

### runTask

```
// runTask runs the given task.
func (w *Worker) runTask(ctx context.Context, task api.AssignedTask) error {
    // Create a sub-context to manage the life-span of both the running of the
    // task and the loop to check whether we're still allowed to run it.
    taskCtx, taskCancel := context.WithCancel(ctx)
    defer taskCancel()

    var taskRunnerErr, abortReason error

    // Run the actual task in a separate goroutine.
    wg := sync.WaitGroup{}
    wg.Add(1)
    go func() {
        defer wg.Done()
        defer taskCancel()
        taskRunnerErr = w.taskRunner.Run(taskCtx, task)
    }()
    ...

    // Wait for the task runner to either complete or abort.
    wg.Wait()
    ...
    return taskRunnerErr
}
```

获取到task之后，在runTask中开一个goroutine执行这个任务，worker中的taskRunner是一个TaskExecutor对象。

TaskExecutor定义如下：

```
type TaskExecutor struct {
    cmdRunner CommandRunner
    listener  TaskExecutionListener
}
```

其中的CommandRunner是一个接口：

```
type CommandRunner interface {
    Run(ctx context.Context, taskID string, cmd api.Command) error
}
```

它是由CommandExecutor实现的，

TaskExecutionListener也是一个接口：

```
// TaskExecutionListener sends task lifecycle events (start/fail/complete) to the Manager.
type TaskExecutionListener interface {
    // TaskStarted tells the Manager that task execution has started.
    TaskStarted(ctx context.Context, taskID string) error

    // TaskFailed tells the Manager the task failed for some reason.
    TaskFailed(ctx context.Context, taskID string, reason string) error

    // TaskCompleted tells the Manager the task has been completed.
    TaskCompleted(ctx context.Context, taskID string) error
}
```

它是由Listener实现。

有关CommandExecutor和Listener的详细功能，在CommandExecutor章节有介绍。