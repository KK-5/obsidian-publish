manager的api_impl模块负责flamenco所提供的API的具体实现，可以视为flamenco向外提供的接口。flamenco中的api都是使用OpenAPI来定义的，关于OpenAPI的相关信息可以查看官方文档： [https://learn.openapis.org/](https://learn.openapis.org/)

## 在pkg中定义API

flamenco的API（REST API）都在pkg模块定义，使用OpenAPI语法，以GetVersion为例，在flamenco-openapi.yaml中定义GetVersion的使用格式：

```
  /api/v3/version:
    summary: Clients can use this to check this is actually a Flamenco server.
    get:
      summary: Get the Flamenco version of this Manager
      operationId: getVersion
      tags: [meta]
      responses:
        "200":
          description: normal response
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/FlamencoVersion"
  ...
  components:
  schemas:
    FlamencoVersion:
      type: object
      properties:
        "version":
          type: string
          description: >
            Version of this Manager, meant for human consumption. For release
            builds it is the same as `shortversion`, for other builds it also
            includes the `git` version info.
        "shortversion": { type: string }
        "name": { type: string }
        "git": { type: string }
      required: [version, shortversion, name, git]
      example:
        version: "3.3-alpha0 (v3.2-76-gdd34d538)"
        shortversion: 3.3-alpha0
        name: Your Manager
        git: v3.2-76-gdd34d538
```

这意味着我们使用http向地址 http://{manager-ip}/api/v3/version发送一个GET方法，就可以获取到flamencao的版本号，它以一个json格式的字符串表示。

同样，在pkg模块中，定义了这个API的响应函数：

```
type ServerInterfaceWrapper struct {
	Handler ServerInterface
}
...
// GetVersion converts echo context to params.
func (w *ServerInterfaceWrapper) GetVersion(ctx echo.Context) error {
	var err error

	// Invoke the callback with all the unmarshalled arguments
	err = w.Handler.GetVersion(ctx)
	return err
}
```

从上面的代码可以看出，真正含有GetVersion方法的类是ServerInterface，而ServerInterfaceWrapper是包装器。pkg模块中的接口都类似一个包装器，用于把各模块对外提供的接口都整合到一起。

## 在api_impl中实现API

接下来就是ServerInterface的实现，它位于api_impl模块的meta.go文件中：

```
func (f *Flamenco) GetVersion(e echo.Context) error {
	return e.JSON(http.StatusOK, api.FlamencoVersion{
		Version:      appinfo.ExtendedVersion(),
		Shortversion: appinfo.ApplicationVersion,
		Name:         f.config.Get().ManagerName,
		Git:          appinfo.ApplicationGitHash,
	})
}
```

GetVersion函数将appinfo和config中的信息读取出来，打包成json格式，并返回出去，对应flamenco-openapi.yaml中声明的查询版本的responses格式。

## Flamenco类

在上面的GetVersion函数中还可以发现，GetVersion其实是Flamenco的成员函数，在Go语言中，如果一个类的成员函数实现了某一个接口的方法，那么这个来就自动实现了这个接口，所以Flamenco就是实现了ServerInterface的类。Flamenco类的定义如下：

```
type Flamenco struct {
	jobCompiler    JobCompiler
	persist        PersistenceService
	broadcaster    ChangeBroadcaster
	logStorage     LogStorage
	config         ConfigService
	stateMachine   TaskStateMachine
	shaman         Shaman
	clock          TimeService
	lastRender     LastRendered
	localStorage   LocalStorage
	sleepScheduler WorkerSleepScheduler
	jobDeleter     JobDeleter

	// The task scheduler can be locked to prevent multiple Workers from getting
	// the same task. It is also used for certain other queries, like
	// `MayWorkerRun` to prevent similar race conditions.
	taskSchedulerMutex sync.Mutex

	// done is closed by Flamenco when it wants the application to shut down and
	// restart itself from scratch.
	done chan struct{}
}

var _ api.ServerInterface = (*Flamenco)(nil)
```

Flamenco包含了很多成员变量，而每一个成员变量就是Manager中的一个功能模块，所以Flamenco类也可以看作是Manager功能的整合。ServerInterface 初始化时就是一个Flamenco类型的空指针。

## Interface

Flamenco类中的每一个成员变量都是一个Interface，负责Manager中的某一个功能，以JobCompiler为例，它的定义如下：

```
type JobCompiler interface {
	ListJobTypes() api.AvailableJobTypes
	GetJobType(typeName string) (api.AvailableJobType, error)
	Compile(ctx context.Context, job api.SubmittedJob) (*job_compilers.AuthoredJob, error)
}
```

它实现3个功能：列出所有类型的Job；查询一个Job的类型；编译Job。当我们需要实现Job编译相关的API时，就可以使用它的功能。以现有的SubmitJob为例，这是最为常用的提交渲染任务的API，它的实现如下：

```
func (f *Flamenco) SubmitJob(e echo.Context) error {
	logger := requestLogger(e)

    // 将提交的任务信息绑定到job变量
	var job api.SubmitJobJSONRequestBody
	if err := e.Bind(&job); err != nil {
		logger.Warn().Err(err).Msg("bad request received")
		return sendAPIError(e, http.StatusBadRequest, "invalid format")
	}

	logger = logger.With().
		Str("type", job.Type).
		Str("name", job.Name).
		Logger()
	logger.Info().Msg("new Flamenco job received")

	ctx := e.Request().Context()
	// 编译这个job，其中使用了JobCompiler中的方法
	authoredJob, err := f.compileSubmittedJob(ctx, logger, api.SubmittedJob(job))
	switch {
	case errors.Is(err, job_compilers.ErrJobTypeBadEtag):
		logger.Info().Err(err).Msg("rejecting submitted job because its settings are outdated, refresh the job type")
		return sendAPIError(e, http.StatusPreconditionFailed, "rejecting job because its settings are outdated, refresh the job type")
	case err != nil:
		logger.Warn().Err(err).Msg("error compiling job")
		// TODO: make this a more specific error object for this API call.
		return sendAPIError(e, http.StatusBadRequest, err.Error())
	}

	logger = logger.With().Str("job_id", authoredJob.JobID).Logger()

	// TODO: check whether this job should be queued immediately or start paused.
	authoredJob.Status = api.JobStatusQueued
    // 将此job储存起来
	if err := f.persist.StoreAuthoredJob(ctx, *authoredJob); err != nil {
		logger.Error().Err(err).Msg("error persisting job in database")
		return sendAPIError(e, http.StatusInternalServerError, "error persisting job in database")
	}

	dbJob, err := f.persist.FetchJob(ctx, authoredJob.JobID)
	if err != nil {
		logger.Error().Err(err).Msg("unable to retrieve just-stored job from database")
		return sendAPIError(e, http.StatusInternalServerError, "error retrieving job from database")
	}
    // 发布job更新消息
	jobUpdate := webupdates.NewJobUpdate(dbJob)
	f.broadcaster.BroadcastNewJob(jobUpdate)

	apiJob := jobDBtoAPI(dbJob)
	return e.JSON(http.StatusOK, apiJob)
}
```

compileSubmittedJob的实现如下：

```
func (f *Flamenco) compileSubmittedJob(ctx context.Context, logger zerolog.Logger, submittedJob api.SubmittedJob) (*job_compilers.AuthoredJob, error) {
	// Replace the special "manager" platform with the Manager's actual platform.
	if submittedJob.SubmitterPlatform == "manager" {
		submittedJob.SubmitterPlatform = runtime.GOOS
	}

	if submittedJob.TypeEtag == nil || *submittedJob.TypeEtag == "" {
		logger.Warn().Msg("job submitted without job type etag, refresh the job types in the Blender add-on")
	}

	// Before compiling the job, replace the two-way variables. This ensures all
	// the tasks also use those.
	replaceTwoWayVariables(f.config, &submittedJob)
    // 在这里调用了JobCompiler中的方法
	return f.jobCompiler.Compile(ctx, submittedJob)
}
```

## 总结

api_impl模块用于实现manager提供的所有API，他把manager上的基础功能整合到一起，提供更易用的接口给开发者。它也是整个manager程序的入口，当启动flamenco-manager程序时，Flamenco类的对象会被创建，其中的各个功能模块开始启动。