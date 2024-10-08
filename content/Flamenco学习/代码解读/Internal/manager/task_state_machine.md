flamenco中的任务有不同的状态，使用状态机对task和job的状态进行管理，在pkg模块下的openapi_types.gen.go文件中有定义二者状态：

```
// Defines values for JobStatus.
const (
	JobStatusActive JobStatus = "active"

	JobStatusCancelRequested JobStatus = "cancel-requested"

	JobStatusCanceled JobStatus = "canceled"

	JobStatusCompleted JobStatus = "completed"

	JobStatusFailed JobStatus = "failed"

	JobStatusPaused JobStatus = "paused"

	JobStatusQueued JobStatus = "queued"

	JobStatusRequeueing JobStatus = "requeueing"

	JobStatusUnderConstruction JobStatus = "under-construction"
)

// Defines values for TaskStatus.
const (
	TaskStatusActive TaskStatus = "active"

	TaskStatusCanceled TaskStatus = "canceled"

	TaskStatusCompleted TaskStatus = "completed"

	TaskStatusFailed TaskStatus = "failed"

	TaskStatusPaused TaskStatus = "paused"

	TaskStatusQueued TaskStatus = "queued"

	TaskStatusSoftFailed TaskStatus = "soft-failed"
)
```

这些状态的定义比较直观。因为一个job可能含有多个task，所以每个job的状态要取决于它所包含的所有task的状态。

### 状态机涉及的前置功能

flamenco中对状态机的管理涉及以下几个功能：

1. 持久化
    

```
type PersistenceService interface {
	SaveTask(ctx context.Context, task *persistence.Task) error
	SaveTaskStatus(ctx context.Context, t *persistence.Task) error
	SaveTaskActivity(ctx context.Context, t *persistence.Task) error
	SaveJobStatus(ctx context.Context, j *persistence.Job) error

	JobHasTasksInStatus(ctx context.Context, job *persistence.Job, taskStatus api.TaskStatus) (bool, error)
	CountTasksOfJobInStatus(ctx context.Context, job *persistence.Job, taskStatuses ...api.TaskStatus) (numInStatus, numTotal int, err error)

	// UpdateJobsTaskStatuses updates the status & activity of the tasks of `job`.
	UpdateJobsTaskStatuses(ctx context.Context, job *persistence.Job,
		taskStatus api.TaskStatus, activity string) error

	// UpdateJobsTaskStatusesConditional updates the status & activity of the tasks of `job`,
	// limited to those tasks with status in `statusesToUpdate`.
	UpdateJobsTaskStatusesConditional(ctx context.Context, job *persistence.Job,
		statusesToUpdate []api.TaskStatus, taskStatus api.TaskStatus, activity string) error

	FetchJobsInStatus(ctx context.Context, jobStatuses ...api.JobStatus) ([]*persistence.Job, error)
	FetchTasksOfWorkerInStatus(context.Context, *persistence.Worker, api.TaskStatus) ([]*persistence.Task, error)
	FetchTasksOfWorkerInStatusOfJob(context.Context, *persistence.Worker, api.TaskStatus, *persistence.Job) ([]*persistence.Task, error)
}
```

由于系统中的job和task都是储存在数据库中的，它们的状态变化都需要即时更新到数据表里。

2. 事件通知系统
    

```
type ChangeBroadcaster interface {
	// BroadcastJobUpdate sends the job update to SocketIO clients.
	BroadcastJobUpdate(jobUpdate api.EventJobUpdate)

	// BroadcastTaskUpdate sends the task update to SocketIO clients.
	BroadcastTaskUpdate(jobUpdate api.EventTaskUpdate)
}
```

task或job的状态变化可能需要通知到其他系统，主要用于web界面的实时显示。

3. 日志储存
    

用于储存日志，可略过。

### 状态机的功能

#### 状态机定义

首先，状态机的定义为：

```
// StateMachine handles task and job status changes.
type StateMachine struct {
	persist     PersistenceService
	broadcaster ChangeBroadcaster
	logStorage  LogStorage
}

func NewStateMachine(persist PersistenceService, broadcaster ChangeBroadcaster, logStorage LogStorage) *StateMachine {
	return &StateMachine{
		persist:     persist,
		broadcaster: broadcaster,
		logStorage:  logStorage,
	}
}
```

非常简单，就是前面三个功能的集成。

#### Task状态变化函数

TaskStatusChange：

```
// TaskStatusChange updates the task's status to the new one.
// `task` is expected to still have its original status, and have a filled `Job` pointer.
func (sm *StateMachine) TaskStatusChange(
	ctx context.Context,
	task *persistence.Task,
	newTaskStatus api.TaskStatus,
) error {
	oldTaskStatus := task.Status

	if err := sm.taskStatusChangeOnly(ctx, task, newTaskStatus); err != nil {
		return err
	}

	if err := sm.updateJobAfterTaskStatusChange(ctx, task, oldTaskStatus); err != nil {
		return fmt.Errorf("updating job after task status change: %w", err)
	}
	return nil
}
```

输入context，task，和newTaskStatus，将task状态置为newTaskStatus，其中包含了两个步骤，一个是task本身的状态变化，一个是这个task状态变化后引起包含了它的job的状态变化。

taskStatusChangeOnly函数：

```
// taskStatusChangeOnly updates the task's status to the new one, but does not "ripple" the change to the job.
// `task` is expected to still have its original status, and have a filled `Job` pointer.
func (sm *StateMachine) taskStatusChangeOnly(
	ctx context.Context,
	task *persistence.Task,
	newTaskStatus api.TaskStatus,
) error {
	job := task.Job
	...
	oldTaskStatus := task.Status
	task.Status = newTaskStatus
	...
	if err := sm.persist.SaveTaskStatus(ctx, task); err != nil {
		return fmt.Errorf("saving task to database: %w", err)
	}
    ...
	// Broadcast this change to the SocketIO clients.
	taskUpdate := eventbus.NewTaskUpdate(task)
	taskUpdate.PreviousStatus = &oldTaskStatus
	sm.broadcaster.BroadcastTaskUpdate(taskUpdate)

	return nil
}
```

这个函数做了两件事：
1. 将task的新状态保存到数据库中。
2. 广播发出一个TaskUpdate事件，主要用于web界面的更新。

updateJobAfterTaskStatusChange函数：

```
// updateJobAfterTaskStatusChange updates the job status based on the status of
// this task and other tasks in the job.
func (sm *StateMachine) updateJobAfterTaskStatusChange(
	ctx context.Context, task *persistence.Task, oldTaskStatus api.TaskStatus,
) error {
	job := task.Job
    ...
	// Every 'case' in this switch MUST return. Just for sanity's sake.
	switch task.Status {
	case api.TaskStatusQueued:
		// Re-queueing a task on a completed job should re-queue the job too.
		return sm.jobStatusIfAThenB(ctx, logger, job, api.JobStatusCompleted, api.JobStatusRequeueing, "task was queued")

	case api.TaskStatusPaused:
		// Pausing a task has no impact on the job.
		return nil

	case api.TaskStatusCanceled:
		return sm.updateJobOnTaskStatusCanceled(ctx, logger, job)

	case api.TaskStatusFailed:
		return sm.updateJobOnTaskStatusFailed(ctx, logger, job)

	case api.TaskStatusActive, api.TaskStatusSoftFailed:
		switch job.Status {
		case api.JobStatusActive, api.JobStatusCancelRequested:
			// Do nothing, job is already in the desired status.
			return nil
		default:
			logger.Info().Msg("job became active because one of its task changed status")
			reason := fmt.Sprintf("task became %s", task.Status)
			return sm.JobStatusChange(ctx, job, api.JobStatusActive, reason)
		}

	case api.TaskStatusCompleted:
		return sm.updateJobOnTaskStatusCompleted(ctx, logger, job)

	default:
		logger.Warn().Msg("task obtained status that Flamenco did not expect")
		return nil
	}
}
```

这里用来处理task状态变化引起的job状态变化，根据变化后的task状态，分为以下情况：

1. Queued：将这个job也置为Queued，等待下次执行。
    
2. Paused：不做任何改变。
    
3. Canceled：查看此时job还有没有其他task需要执行，如果有，job状态不发生变化，如果没有，job状态也变为Canceled。
    
4. Failed：查看job中failed的task的数量，如果大于某一个阈值（默认是10%），则此job状态变为Failed，如果小于此阈值，且此时job状态为Queued，那么job状态变为Active，否则不做变化。
    
5. Active和SoftFailed：查看此时job的状态，如果job状态为Active或CancelRequest，则不做变化，否则job变化为Active状态。
    
6. Completed：查看job中的其他所有task状态，如果全部Completed，则job的状态也变为Completed，如果没有全部完成，且此时job状态为Queued，job状态变化为Active。
    

#### Job状态变化函数

JobStatusChange ：

```
// JobStatusChange gives a Job a new status, and handles the resulting status changes on its tasks.
func (sm *StateMachine) JobStatusChange(
	ctx context.Context,
	job *persistence.Job,
	newJobStatus api.JobStatus,
	reason string,
) error {
	// Job status changes can trigger task status changes, which can trigger the
	// next job status change. Keep looping over these job status changes until
	// there is no more change left to do.
	var err error
	for newJobStatus != "" && newJobStatus != job.Status {
		oldJobStatus := job.Status
		job.Activity = fmt.Sprintf("Changed to status %q: %s", newJobStatus, reason)
        ...
		newJobStatus, err = sm.jobStatusSet(ctx, job, newJobStatus, reason, logger)
		if err != nil {
			return err
		}
	}

	return nil
}
```

这个状态设置函数会一直调用jobStatusSet去设置job的状态，直到新状态为空或者与当前状态一致，这说明，为job设置一个状态时，可能会引起不止一次的job状态改变。

jobStatusSet：

```
// jobStatusSet saves the job with the new status and handles updates to tasks
// as well. If the task status change should trigger another job status change,
// the new job status is returned.
func (sm *StateMachine) jobStatusSet(ctx context.Context,
    job *persistence.Job,
    newJobStatus api.JobStatus,
    reason string,
    logger zerolog.Logger,
) (api.JobStatus, error) {
    oldJobStatus := job.Status
    job.Status = newJobStatus

    // Persist the new job status.
    err := sm.persist.SaveJobStatus(ctx, job)
    if err != nil {
        return "", fmt.Errorf("saving job status change %q to %q to database: %w",
            oldJobStatus, newJobStatus, err)
    }

    // Handle the status change.
    result, err := sm.updateTasksAfterJobStatusChange(ctx, logger, job, oldJobStatus)
    if err != nil {
        return "", fmt.Errorf("updating job's tasks after job status change: %w", err)
    }

    // Broadcast this change to the SocketIO clients.
    jobUpdate := eventbus.NewJobUpdate(job)
    jobUpdate.PreviousStatus = &oldJobStatus
    jobUpdate.RefreshTasks = result.massTaskUpdate
    sm.broadcaster.BroadcastJobUpdate(jobUpdate)

    return result.followingJobStatus, nil
}
```

这个函数的注释中说明了，job的状态改变会导致它包含的task的状态改变，如果这些task状态的改变继续影响了job，那么这个函数会返回这个job状态，这也是修改状态时需要循环尝试的原因。

job状态改变包含3个步骤：

1. 将job设置为新状态并保存到数据库中。
    
2. 更新job状态引起的task状态变化，这个过程中可能又会改变job的状态。
    
3. 发送一个JobUpdate事件。
    

updateTasksAfterJobStatusChange：

```
// updateTasksAfterJobStatusChange updates the status of its tasks based on the
// new status of this job.
//
// NOTE: this function assumes that the job already has its new status.
//
// Returns the new state the job should go into after this change, or an empty
// string if there is no subsequent change necessary.
func (sm *StateMachine) updateTasksAfterJobStatusChange(
    ctx context.Context,
    logger zerolog.Logger,
    job *persistence.Job,
    oldJobStatus api.JobStatus,
) (tasksUpdateResult, error) {

    // Every case in this switch MUST return, for sanity sake.
    switch job.Status {
    case api.JobStatusCompleted, api.JobStatusCanceled:
        // Nothing to do; this will happen as a response to all tasks receiving this status.
        return tasksUpdateResult{}, nil

    case api.JobStatusActive:
        // Nothing to do; this happens when a task gets started, which has nothing to
        // do with other tasks in the job.
        return tasksUpdateResult{}, nil

    case api.JobStatusCancelRequested, api.JobStatusFailed:
        jobStatus, err := sm.cancelTasks(ctx, logger, job)
        return tasksUpdateResult{
            followingJobStatus: jobStatus,
            massTaskUpdate:     true,
        }, err

    case api.JobStatusRequeueing:
        jobStatus, err := sm.requeueTasks(ctx, logger, job, oldJobStatus)
        return tasksUpdateResult{
            followingJobStatus: jobStatus,
            massTaskUpdate:     true,
        }, err

    case api.JobStatusQueued:
        jobStatus, err := sm.checkTaskCompletion(ctx, logger, job)
        return tasksUpdateResult{
            followingJobStatus: jobStatus,
            massTaskUpdate:     true,
        }, err

    default:
        logger.Warn().Msg("unknown job status change, ignoring")
        return tasksUpdateResult{}, nil
    }
}
```

首先，这个函数返回的是一个tasksUpdateResult结构体，它包含了两个元素，，一个是job变化后的新状态，一个控制是否更新job下的所有task。上面的JobStatusChange 函数就会用这个新状态与应该设置的状态比较，直到它们一致。

根据job设置的状态，对task的影响有以下情况：

1. Completed，Canceled，Active：对task没有任何影响。
    
2. CancelRequested，Failed：将job下所有状态为Active，Queued，SoftFailed的task全部置为Canceled，然后如果job状态是CancelRequested，它的下一个状态变为Canceled。
    
3. Requeueing：  
    根据job之前的状态：
    
    1. UnderConstruction：不做任何改变。
        
    2. Completed：将所有任务都置为Queued重新执行，job下一状态变为Queued
        
    3. 其他状态：将job下所有状态为Canceled，Failed，Paused，SoftFailed的task全部置为Queued重新执行，job下一状态变为Queued。
        
4. Queued：检测这个job下的所有task，如果全部为Completed，job的状态也变为Completed，否则不做变化。
    

#### Task在worker上的重新执行

requeueTasksOfWorker：

```
func (sm *StateMachine) requeueTasksOfWorker(
	ctx context.Context,
	tasks []*persistence.Task,
	worker *persistence.Worker,
	reason string,
) error {
    ...
	// Run each task change through the task state machine.
	var lastErr error
	for _, task := range tasks {
        ...
			// Write to task activity that it got requeued because of worker sign-off.
		task.Activity = "Task was requeued by Manager because " + reason
		if err := sm.persist.SaveTaskActivity(ctx, task); err != nil {
            ...
			lastErr = err
		}

		if err := sm.TaskStatusChange(ctx, task, api.TaskStatusQueued); err != nil {
            ...
			lastErr = err
		}

		// The error is already logged by the log storage.
		_ = sm.logStorage.WriteTimestamped(logger, task.Job.UUID, task.UUID, task.Activity)
	}

	return lastErr
}
```

这个函数可以把一个worker上的task重新加入队列执行，针对每一个任务，先将它们的状态都置为active，然后改变状态为Queued，借助这个函数，状态机提供了两个函数用来重新执行task：

RequeueActiveTasksOfWorker：将所有Active状态的任务重新加入队列，应该是用于任务的重新排列。

RequeueFailedTasksOfWorkerOfJob：将所有失败的任务重新加入队列执行。

### 总结

flamenco使用状态机来管理job和task，状态机模块提供了修改二者状态的接口，并且在其中维护了一整套状态变化的逻辑，其中的job和task信息都保存在数据库中，包括它们的状态，其他的模块可以读取job或task的当前状态来决定如何处理这些任务。