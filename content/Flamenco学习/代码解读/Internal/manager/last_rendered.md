此模块用于处理最后一次的渲染结果并提供预览图片在web端显示，它提供的接口如下：

```
type LastRendered interface {
	// QueueImage queues an image for processing. Returns
	// `last_rendered.ErrQueueFull` if there is no more space in the queue for
	// new images.
	QueueImage(payload last_rendered.Payload) error

	// PathForJob returns the base path for this job's last-rendered images.
	PathForJob(jobUUID string) string

	// ThumbSpecs returns the thumbnail specifications.
	ThumbSpecs() []last_rendered.Thumbspec

	// JobHasImage returns true only if the job actually has a last-rendered image.
	JobHasImage(jobUUID string) bool
}
```

## image_processing

使用go语言中的image库来处理图片，包括图片的解码，压缩，保存。

## last_rendered

last_rendered中的数据结构定义如下：

```
type Storage interface {
	// ForJob returns the directory path for storing job-related files.
	ForJob(jobUUID string) string
}

// LastRenderedProcessor processes "last-rendered" images and stores them with
// the job.
type LastRenderedProcessor struct {
	storage Storage

	// TODO: expand this queue to be per job, so that one spammy job doesn't block
	// the queue for other jobs.
	queue chan Payload
}

// Payload contains the actual image to process.
type Payload struct {
	JobUUID    string // Used to determine the directory to store the image.
	WorkerUUID string // Just for logging.
	MimeType   string
	Image      []byte

	// Callback is called when the image processing is finished.
	Callback func(ctx context.Context)
}

// Thumbspec specifies a thumbnail size & filename.
type Thumbspec struct {
	Filename  string
	MaxWidth  int
	MaxHeight int
}
```

每个数据结构的功能已经注释已经介绍清楚了。需要处理的图片信息保存到一个Payload 中，LastRenderedProcessor 有一个Payload 队列，它会循环处理队列中的每一张图片。

看一下四个接口的实现：

```
// 将一个Payload加入到LastRenderedProcessor的队列中
// QueueImage queues an image for processing.
// Returns `ErrQueueFull` if there is no more space in the queue for new images.
func (lrp *LastRenderedProcessor) QueueImage(payload Payload) error {
	logger := payload.sublogger(log.Logger)
	select {
	case lrp.queue <- payload:
		logger.Debug().Msg("last-rendered: queued image for processing")
		return nil
	default:
		logger.Debug().Msg("last-rendered: unable to queue image for processing")
		return ErrQueueFull
	}
}
```

```
// 简单的storage功能包装，输入一个job的uuid，输出它的last-rendered images
// PathForJob returns the base path for this job's last-rendered images.
func (lrp *LastRenderedProcessor) PathForJob(jobUUID string) string {
	return lrp.storage.ForJob(jobUUID)
}
```

```
// 判断job是否有last-rendered image
// JobHasImage returns true only if the job actually has a last-rendered image.
// Only the lowest-resolution image is tested for. Since images are processed in
// order, existence of the last one should imply existence of all of them.
func (lrp *LastRenderedProcessor) JobHasImage(jobUUID string) bool {
	dirPath := lrp.PathForJob(jobUUID)
	filename := thumbnails[len(thumbnails)-1].Filename
	path := filepath.Join(dirPath, filename)

	_, err := os.Stat(path)
	switch {
	case err == nil:
		return true
	case errors.Is(err, fs.ErrNotExist):
		return false
	default:
		log.Warn().Err(err).Str("path", path).Msg("last-rendered: unexpected error checking file for existence")
		return false
	}
}
```

```
// 获得缩略图的配置
	thumbnails = []Thumbspec{
		{"last-rendered.jpg", 1920, 1080},
		{"last-rendered-small.jpg", 600, 338},
		{"last-rendered-tiny.jpg", 200, 112},
	}

// ThumbSpecs returns the thumbnail specifications.
func (lrp *LastRenderedProcessor) ThumbSpecs() []Thumbspec {
	// Return a copy so modification of the returned slice won't affect the global
	// `thumbnails` variable.
	copied := make([]Thumbspec, len(thumbnails))
	copy(copied, thumbnails)
	return copied
}
```

处理图片的核心函数：

```
// processImage down-scales the image to a few thumbnails for presentation in
// the web interface, and stores those in a job-specific directory.
//
// Because this is intended as internal queue-processing function, errors are
// logged but not returned.
func (lrp *LastRenderedProcessor) processImage(ctx context.Context, payload Payload) {
	jobDir := lrp.PathForJob(payload.JobUUID)

	logger := log.With().Str("jobDir", jobDir).Logger()
	logger = payload.sublogger(logger)

	// Decode the image.
	image, err := decodeImage(payload)
	if err != nil {
		logger.Error().Err(err).Msg("last-rendered: unable to decode image")
		return
	}

	// Generate the thumbnails.
	for _, spec := range thumbnails {
		thumbLogger := spec.sublogger(logger)
		thumbLogger.Trace().Msg("last-rendered: creating thumbnail")
        // 将图片缩小
		image = downscaleImage(spec, image)
        // 保存在对应的job下面
		imgpath := filepath.Join(jobDir, spec.Filename)
		if err := saveJPEG(imgpath, image); err != nil {
			thumbLogger.Error().Err(err).Msg("last-rendered: error saving thumbnail")
			break
		}
	}

	// Call the callback, if provided.
	if payload.Callback != nil {
		payload.Callback(ctx)
	}
}
```

## 总结

last_rendered是一个工具模块，用于将渲染出的图片生成预览图，这样渲染完成后就可以在web端直接看到渲染结果，总体上来说是一个为了程序的易用性设计的模块。