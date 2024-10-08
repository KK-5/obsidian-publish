persistence是manager中最复杂的一个模块，复制manager数据的持久化工作，它提供了很多持久化数据的接口给上层应用，比如worker信息的保存、更新，job信息的创建、保存、状态更新、删除等。

manager中使用spllite作为持久化的工具，同时使用gorm库来访问数据库。gorm库是go语言中的一个进行ORM的库，可以以go语言的形式来进行数据库的访问，而不需要编写大量的sql语句，关于gorm的详细信息可以查看 [https://gorm.io/docs/](https://gorm.io/docs/) 。

## DB

DB是persistence模块中的数据库对象，persistence的接口都由它来提供，它的定义如下：

```
// DB provides the database interface.
type DB struct {
	gormDB *gorm.DB
}

// Model contains the common database fields for most model structs.
// It is a copy of the gorm.Model struct, but without the `DeletedAt` field.
// Soft deletion is not used by Flamenco. If it ever becomes necessary to
// support soft-deletion, see https://gorm.io/docs/delete.html#Soft-Delete
type Model struct {
	ID        uint `gorm:"primarykey"`
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

DB包含了一个*gorm.DB类型的结构体，Model 是数据库中所有model结构的公共字段。

OpenDB(ctx context.Context, dsn string) (*DB, error)用于创建DB，核心代码：

```
	...
	dialector := sqlite.Open(dsn)
	gormDB, err := gorm.Open(dialector, config)
	if err != nil {
		return nil, err
	}

	db := DB{
		gormDB: gormDB,
	}
	...
```

(db *DB) Close() error用于关闭DB连接：

```
// Close closes the connection to the database.
func (db *DB) Close() error {
	sqldb, err := db.gormDB.DB()
	if err != nil {
		return err
	}
	return sqldb.Close()
}
```

当DB创建之后，自动为数据库关联以下model，

```
	err := db.gormDB.AutoMigrate(
		&Job{},
		&JobBlock{},
		&JobStorageInfo{},
		&LastRendered{},
		&SleepSchedule{},
		&Task{},
		&TaskFailure{},
		&Worker{},
		&WorkerTag{},
	)
```

这些就是数据库中包含的表结构，目前它们没有任何字段。

## jobs

jobs专门用于对flamenco中job的持久化，包括储存、更新、删除和查找等，它涉及的model如下：

```
type Job struct {
	Model  // Model是所有表包含的公共字段
	UUID string `gorm:"type:char(36);default:'';unique;index"`

	Name     string        `gorm:"type:varchar(64);default:''"`
	JobType  string        `gorm:"type:varchar(32);default:''"`
	Priority int           `gorm:"type:smallint;default:0"`
	Status   api.JobStatus `gorm:"type:varchar(32);default:''"`
	Activity string        `gorm:"type:varchar(255);default:''"`

	Settings StringInterfaceMap `gorm:"type:jsonb"`
	Metadata StringStringMap    `gorm:"type:jsonb"`

	DeleteRequestedAt sql.NullTime

	Storage JobStorageInfo `gorm:"embedded;embeddedPrefix:storage_"`

	WorkerTagID *uint
	WorkerTag   *WorkerTag `gorm:"foreignkey:WorkerTagID;references:ID;constraint:OnDelete:SET NULL"`
}

type Task struct {
	Model
	UUID string `gorm:"type:char(36);default:'';unique;index"`

	Name     string         `gorm:"type:varchar(64);default:''"`
	Type     string         `gorm:"type:varchar(32);default:''"`
	JobID    uint           `gorm:"default:0"`
	Job      *Job           `gorm:"foreignkey:JobID;references:ID;constraint:OnDelete:CASCADE"`
	Priority int            `gorm:"type:smallint;default:50"`
	Status   api.TaskStatus `gorm:"type:varchar(16);default:''"`

	// Which worker is/was working on this.
	WorkerID      *uint
	Worker        *Worker   `gorm:"foreignkey:WorkerID;references:ID;constraint:OnDelete:SET NULL"`
	LastTouchedAt time.Time `gorm:"index"` // Should contain UTC timestamps.

	// Dependencies are tasks that need to be completed before this one can run.
	Dependencies []*Task `gorm:"many2many:task_dependencies;constraint:OnDelete:CASCADE"`

	Commands Commands `gorm:"type:jsonb"`
	Activity string   `gorm:"type:varchar(255);default:''"`
}

// TaskFailure keeps track of which Worker failed which Task.
type TaskFailure struct {
	// Don't include the standard Gorm ID, UpdatedAt, or DeletedAt fields, as they're useless here.
	// Entries will never be updated, and should never be soft-deleted but just purged from existence.
	CreatedAt time.Time
	TaskID    uint    `gorm:"primaryKey;autoIncrement:false"`
	Task      *Task   `gorm:"foreignkey:TaskID;references:ID;constraint:OnDelete:CASCADE"`
	WorkerID  uint    `gorm:"primaryKey;autoIncrement:false"`
	Worker    *Worker `gorm:"foreignkey:WorkerID;references:ID;constraint:OnDelete:CASCADE"`
}
```

以保存job的状态为例，它的代码如下：

```
// SaveJobStatus saves the job's Status and Activity fields.
func (db *DB) SaveJobStatus(ctx context.Context, j *Job) error {
	tx := db.gormDB.WithContext(ctx).
		Model(j).
		Updates(Job{Status: j.Status, Activity: j.Activity})
	if tx.Error != nil {
		return jobError(tx.Error, "saving job status")
	}
	return nil
}
```

SaveJobStatus会更新传入job的Status和Activity两个字段。

其余的增删查改逻辑基本类似，不再详细介绍。jobs提供了以下持久化的接口：

- StoreAuthoredJob：保存编译好的job。
    
- FetchJob：通过jobID获取一个job。
    
- SaveJobPriority：保存（更新）job的优先级。
    
- FetchTask：通过taskID获取一个task。
    
- FetchTaskFailureList：获取执行某个task失败的所有worker。
    
- SaveTask：保存task。
    
- SaveTaskActivity：保存task的active状态。
    
- TaskTouchedByWorker：将一个task标记为’touched‘，这个操作发生在worker上，用于超时检测。
    
- QueryJobs：查询符合条件的所有job。
    
- QueryJobTaskSummaries：查询一个job下的所有task。
    

## workers

workers用于manager对连接到它的所有worker进行数据管理，manager上会储存worker的信息，以便于对worker进行控制（让一个worker休眠），或者为worker分配job。

workers所涉及的model：

```
type Worker struct {
	Model
	DeletedAt gorm.DeletedAt `gorm:"index"`

	UUID   string `gorm:"type:char(36);default:'';unique;index"`
	Secret string `gorm:"type:varchar(255);default:''"`
	Name   string `gorm:"type:varchar(64);default:''"`

	Address    string           `gorm:"type:varchar(39);default:'';index"` // 39 = max length of IPv6 address.
	Platform   string           `gorm:"type:varchar(16);default:''"`
	Software   string           `gorm:"type:varchar(32);default:''"`
	Status     api.WorkerStatus `gorm:"type:varchar(16);default:''"`
	LastSeenAt time.Time        `gorm:"index"` // Should contain UTC timestamps.

	StatusRequested   api.WorkerStatus `gorm:"type:varchar(16);default:''"`
	LazyStatusRequest bool             `gorm:"type:smallint;default:false"`

	SupportedTaskTypes string `gorm:"type:varchar(255);default:''"` // comma-separated list of task types.

	Tags []*WorkerTag `gorm:"many2many:worker_tag_membership;constraint:OnDelete:CASCADE"`
}

type WorkerTag struct {
	Model

	UUID        string `gorm:"type:char(36);default:'';unique;index"`
	Name        string `gorm:"type:varchar(64);default:'';unique"`
	Description string `gorm:"type:varchar(255);default:''"`

	Workers []*Worker `gorm:"many2many:worker_tag_membership;constraint:OnDelete:CASCADE"`
}
```

提供的持久化接口：

- CreateWorker：创建一个worker记录。
    
- FetchWorker：通过uuid查找worker。
    
- FetchWorkers：查找连接到此manager的所有worker。
    
- FetchWorkerTask：获得最近被指定到此worker上的task。
    
- SaveWorker：保存（更新）worker记录。
    
- SaveWorkerStatus：保存（更新）worker状态。
    
- WorkerSeen：将一个worker标记为此manager可见，用于超时检测。
    
- DeleteWorker：通过uuid删除一个worker。
    

除了jobs和workers这两个主要模块，persistence还提供了一些其余的持久化接口，比如last_render图片的保存。就实现逻辑而言，基本都是不同表之间的连接以及增删查改，因此这里只描述每个接口的功能，不详细研究他们的实现方式。