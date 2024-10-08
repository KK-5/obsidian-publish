job_compilers模块的主要功能是支持用户自定义flamenco上的job，flamenco默认有一个渲染场景的job类型供用户使用，但是也支持用户根据自身的需要编写job。自定义的job使用javascript来编码，将其放在manager程序文件夹下的script目录中（没有此目录时可以自己创建），manager会自动识别并编译这些job，后续就可以通过API来使用它们了。

关于job的详细介绍，可参考：[https://flamenco.blender.org/usage/job-types/](https://flamenco.blender.org/usage/job-types/)

job_compilers模块提供以下3个功能：

```
type JobCompiler interface {
	ListJobTypes() api.AvailableJobTypes
	GetJobType(typeName string) (api.AvailableJobType, error)
	Compile(ctx context.Context, job api.SubmittedJob) (*job_compilers.AuthoredJob, error)
}
```

## goja

job使用javascript编写的，flamenco主要使用go编写，所以需要一种让两种编程语言相互协同的手段，flamenco使用的时goja库。goja 包提供了一套完整的 JavaScript 运行环境，包括了解析器、虚拟机、标准库等，可以在 Go 程序中通过直接调用函数或间接调用脚本来执行 JavaScript 代码。goja 还支持绑定和导出本地 Go 函数或类型，使 JavaScript 代码能够与 Go 代码进行交互。 关于goja库更详细的介绍可以查看：[https://pkg.go.dev/github.com/dop251/goja#section-documentation](https://pkg.go.dev/github.com/dop251/goja#section-documentation)

## scripts.go

script.go负责从磁盘中读取job文件，并创建javascript运行所需的虚拟机环境。

job_compilers中有如下数据类型定义：

```
// Service contains job compilers defined in JavaScript.
type Service struct {
	compilers   map[string]Compiler // Mapping from job type name to the job compiler of that type.
	registry    *require.Registry   // Goja module registry.
	timeService TimeService

	// mutex protects 'compilers' from race conditions.
	mutex *sync.Mutex
}

type Compiler struct {
	jobType  string
	program  *goja.Program // Compiled JavaScript file.
	filename string        // The filename of that JS file.
}

type VM struct {
	runtime     *goja.Runtime // Goja VM containing the job compiler script.
	compiler    Compiler      // Program loaded into this VM.
	jobTypeEtag string        // Etag for this particular job type.
}

// jobCompileFunc is a function that fills job.Tasks.
type jobCompileFunc func(job *AuthoredJob) error
```

Service是job_campilers的核心，包含所有job的编译器，VM是job执行的虚拟机，它们都包含了Compiler结构体。Compiler中的program就是定义job的javascript文件编译而来的代码。script.go主要实现两个函数：

- loadScripts()：从文件夹中读取job文件，并将它的compile函数储存到 Service.compilers 中。
    
- compilerVMForJobType(jobTypeName string)：输入一个job类型，返回该job运行的虚拟机环境，即上面的VM结构体。
    

在compilerVMForJobType函数中，有这样一段代码：

```
	runtime := newGojaVM(s.registry)
	if _, err := runtime.RunProgram(program.program); err != nil {
		return nil, err
	}
```

他调用newGojaVM函数创建了一个goja虚拟机，用于初始化最终返回的VM结构体。newGojaVM函数的实现如下：

```
func newGojaVM(registry *require.Registry) *goja.Runtime {
	vm := goja.New()
	vm.SetFieldNameMapper(goja.UncapFieldNameMapper())

	mustSet := func(name string, value interface{}) {
		err := vm.Set(name, value)
		if err != nil {
			log.Panic().Err(err).Msgf("unable to register '%s' in Goja VM", name)
		}
	}

	// Set some global functions.
	mustSet("print", jsPrint)
	mustSet("alert", jsAlert)
	mustSet("frameChunker", jsFrameChunker)
	mustSet("formatTimestampLocal", jsFormatTimestampLocal)

	// Pre-import some useful modules.
	registry.Enable(vm)
	mustSet("author", require.Require(vm, "author"))
	mustSet("path", require.Require(vm, "path"))
	mustSet("process", require.Require(vm, "process"))

	return vm
}
```

可以看到，在创建goja的虚拟机时，这里向其中注入了几个方法，比如用于调试的jsPrint，在虚拟机中的名称为print，这就表示在使用javascript编写job时，可以使用print函数来间接使用jsPrint打印信息，其他几个函数也是同理。除了函数外，这里也注入了几个模块，也可以理解成类，path模块提供了manager运行的路径信息，process模块提供了当前路径和当前的运行平台（windows, linux），author模块则是比较核心的模块，接下来详细介绍。

## author

author模块专门提供一些方法让用户来自定义job，是job编写中最常使用的模块。

author中有如下数据结构定义：

```
// Author allows scripts to author tasks and commands.
type Author struct {
	runtime *goja.Runtime
}

// AuthoredJob就是自定义的Job，它包含了多个Task
type AuthoredJob struct {
	JobID         string
	WorkerTagUUID string

	Name     string
	JobType  string
	Priority int
	Status   api.JobStatus

	Created time.Time

	Settings JobSettings
	Metadata JobMetadata
	Storage  JobStorageInfo

	Tasks []AuthoredTask
}
type JobSettings map[string]interface{}
type JobMetadata map[string]string

// AuthoredTask是Job中具体执行的任务，任务的执行命令保存在Commands中
type AuthoredTask struct {
	// Tasks already get their UUID in the authoring stage. This makes it simpler
	// to store the dependencies, as the code doesn't have to worry about value
	// vs. pointer semantics. Tasks can always be unambiguously referenced by
	// their UUID.
	UUID     string
	Name     string
	Type     string
	Priority int
	Commands []AuthoredCommand

	// Dependencies are tasks that need to be completed before this one can run.
	Dependencies []*AuthoredTask `json:"omitempty" yaml:"omitempty"`
}

type AuthoredCommand struct {
	Name       string
	Parameters AuthoredCommandParameters
}
```

author模块向两个javascript提供了两个函数，Task和Command，用来在自定以Job时创建Task和它所执行的命令。两个函数的具体实现：

```
// 输入任务的名称和类型，返回AuthoredTask
func (a *Author) Task(name string, taskType string) (*AuthoredTask, error) {
	name = strings.TrimSpace(name)
	taskType = strings.TrimSpace(taskType)
	if name == "" {
		return nil, errors.New("author.Task(name, type): name is required")
	}
	if taskType == "" {
		return nil, errors.New("author.Task(name, type): type is required")
	}

	at := AuthoredTask{
		uuid.New(),
		name,
		taskType,
		50, // TODO: handle default priority somehow.
		make([]AuthoredCommand, 0), // 创建的是一个没有command的空任务
		make([]*AuthoredTask, 0),
	}
	return &at, nil
}
```

```
// 输入需要执行的cmd命令和参数，返回一个Command
func (a *Author) Command(cmdName string, parameters AuthoredCommandParameters) (*AuthoredCommand, error) {
	ac := AuthoredCommand{cmdName, parameters}
	return &ac, nil
}
```

除此之外，author模块也提供了AddTask（将一个AuthoredTask加入到AuthoredJob）、AddCommand（将一个Command加入到AuthoredTask中），AddDependency（将一个AuthoredTask加入到另一个AuthoredTask的依赖中）三个函数，加上上面的Task和Command函数，自定义job所需的接口就全部准备好了，后面三个add函数的实现比较简单，append到对应的数据中就行，不再详细分析。

## job_compilers

job_compilers用于整合上述的所有模块，实现自定义job的加载和编译服务，并实现此模块需要提供的三个接口（ListJobTypes()，GetJobType(typeName string)，Compile(ctx context.Context, job api.SubmittedJob)）。前两个接口比较简单，这里以Compile为例来分析。先看Compile接口的实现：

```
func (s *Service) Compile(ctx context.Context, sj api.SubmittedJob) (*AuthoredJob, error) {
	// 获得该job类型的VM
	vm, err := s.compilerVMForJobType(sj.Type)
	if err != nil {
		return nil, err
	}

	if err := vm.checkJobTypeEtag(sj); err != nil {
		return nil, err
	}

	// Create an AuthoredJob from this SubmittedJob.
	// 注意这里的Settings和Metadata是空的
	aj := AuthoredJob{
		JobID:    uuid.New(),
		Created:  s.timeService.Now(),
		Name:     sj.Name,
		JobType:  sj.Type,
		Priority: sj.Priority,
		Status:   api.JobStatusUnderConstruction,

		Settings: make(JobSettings),
		Metadata: make(JobMetadata),
	}
	if sj.Settings != nil {
		for key, value := range sj.Settings.AdditionalProperties {
			aj.Settings[key] = value
		}
	}
	if sj.Metadata != nil {
		for key, value := range sj.Metadata.AdditionalProperties {
			aj.Metadata[key] = value
		}
	}

	if sj.Storage != nil && sj.Storage.ShamanCheckoutId != nil {
		aj.Storage.ShamanCheckoutID = *sj.Storage.ShamanCheckoutId
	}

	if sj.WorkerTag != nil {
		aj.WorkerTagUUID = *sj.WorkerTag
	}
    // 从VM中获取该job的compiler，它们在manager启动时就加载好了
	compiler, err := vm.getCompileJob()
	if err != nil {
		return nil, err
	}
	// 调用这个compiler函数编译该job,主要为填充job的task和command
	if err := compiler(&aj); err != nil {
		return nil, err
	}

	log.Info().
		Int("num_tasks", len(aj.Tasks)).
		Str("name", aj.Name).
		Str("jobtype", aj.JobType).
		Str("job", aj.JobID).
		Msg("job compiled")

	return &aj, nil
}
```

vm.getCompileJob()的实现如下：

```
func (vm *VM) getCompileJob() (jobCompileFunc, error) {
    // 所谓的compiler由用户自定义，在javascript中名称为compileJob
	compileJob, isCallable := goja.AssertFunction(vm.runtime.Get("compileJob"))
	if !isCallable {
		// TODO: construct a more elaborate Error type that contains this info, instead of logging here.
		log.Error().
			Str("jobType", vm.compiler.jobType).
			Str("script", vm.compiler.filename).
			Msg("script does not define a compileJob(job) function")
		return nil, ErrScriptIncomplete
	}

	// TODO: wrap this in a nicer way.
	return func(job *AuthoredJob) error {
		_, err := compileJob(nil, vm.runtime.ToValue(job))
		return err
	}, nil
}
```

Compile的功能可以总结为：当用户提交job时，通过该job的类型寻找对应的compiler（这个job的类型可能是manager自带的，也可能是用户自己定义的，其中包括了该类型的任务的参和compiler），使用该compiler构建好job的task和对应的command，封装到一个完整的AuthoredJob中，后面就可以发送给worker执行了。

## 一个Job_type示例

以manager中自带的echo_and_sleep的job类型举例，它的定义如下：

```
const JOB_TYPE = {
    label: "Echo Sleep Test",  // job type 的名称
    settings: [                // 对应JobSettings
        { key: "message", type: "string", required: true },
        { key: "sleep_duration_seconds", type: "int32", default: 1 },
        { key: "sleep_repeats", type: "int32", default: 1 },
    ]
};


function compileJob(job) {   // 该job type的compiler，名称必须为compileJob，接受AuthoredJob参数
    const settings = job.settings;

    const echoTask = author.Task("echo", "misc"); // 使用author的Task函数创建一个task，它的名称为scho，类型为misc
    echoTask.addCommand(author.Command("echo", {message: settings.message})); // 使用addCommand为这个task添加echo命令
    job.addTask(echoTask);  // 将这个task加入job
    // 同理，加入sleep的task
    for (let repeat=0; repeat < settings.sleep_repeats; repeat++) {
      const sleepTask = author.Task("sleep", "misc")
      sleepTask.addCommand(author.Command("sleep", {duration_in_seconds: settings.sleep_duration_seconds}))
      sleepTask.addDependency(echoTask); // Ensure sleeping happens after echo, and not at the same time.
      job.addTask(sleepTask);
    }
}
```