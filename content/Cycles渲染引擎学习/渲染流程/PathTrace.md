PathTrace是路径追踪模块，包含了PathTraceWork，负责整个路径追踪算法的调度和执行。根据官方代码注释PathTrace负责的工作包括但不限于以下几种：Kernel graph, Scheduling logic, Queues management,  Adaptive stopping。具体含义是：kernel算法图的维护（执行的kernel和顺序），逻辑调度，DeviceQueued的管理，针对场景的自适应停止算法。

## 构造函数
```
PathTrace::PathTrace(Device *device,
                     Film *film,
                     DeviceScene *device_scene,
                     RenderScheduler &render_scheduler,
                     TileManager &tile_manager)
    : device_(device),
      film_(film),
      device_scene_(device_scene),
      render_scheduler_(render_scheduler),
      tile_manager_(tile_manager)
{
  DCHECK_NE(device_, nullptr);

  {
    vector<DeviceInfo> cpu_devices;
    device_cpu_info(cpu_devices);

    cpu_device_.reset(device_cpu_create(cpu_devices[0], device->stats, device->profiler));
  }

  /* Create path tracing work in advance, so that it can be reused by incremental sampling as much
   * as possible. */
  device_->foreach_device([&](Device *path_trace_device) {
    unique_ptr<PathTraceWork> work = PathTraceWork::create(
        path_trace_device, film, device_scene, &render_cancel_.is_requested);
    if (work) {
      path_trace_works_.emplace_back(std::move(work));
    }
  });

  work_balance_infos_.resize(path_trace_works_.size());
  work_balance_do_initial(work_balance_infos_);

  render_scheduler.set_need_schedule_rebalance(path_trace_works_.size() > 1);
}
```

PathTrace在构造时自动根据每个device创建PathTraceWork（PathTraceWork不能实例化，只能通过这个create函数构造）。

## 成员变量

```
  /* Pointer to a device which is configured to be used for path tracing. If multiple devices
   * are configured this is a `MultiDevice`. */
  Device *device_ = nullptr;

  /* CPU device for creating temporary render buffers on the CPU side. */
  // 这个CPU设备并不是指渲染设备
  unique_ptr<Device> cpu_device_;

  Film *film_;
  DeviceScene *device_scene_;

  RenderScheduler &render_scheduler_;
  TileManager &tile_manager_;

  /* Display driver for interactive render buffer display. */
  unique_ptr<PathTraceDisplay> display_;

  /* Output driver to write render buffer to. */
  unique_ptr<OutputDriver> output_driver_;

  /* Per-compute device descriptors of work which is responsible for path tracing on its configured
   * device. */
  // 所有渲染工作
  vector<unique_ptr<PathTraceWork>> path_trace_works_;

  /* Per-path trace work information needed for multi-device balancing. */
  vector<WorkBalanceInfo> work_balance_infos_;

  /* Render buffer parameters of the full frame and current big tile. */
  // buffer的参数
  BufferParams full_params_;
  BufferParams big_tile_params_;

  /* Denoiser which takes care of denoising the big tile. */
  unique_ptr<Denoiser> denoiser_;

  /* Denoiser device descriptor which holds the denoised big tile for multi-device workloads. */
  unique_ptr<PathTraceWork> big_tile_denoise_work_;
```

## 相关的数据结构

### RenderWork

```
class RenderWork {
 public:
  int resolution_divider = 1;

  /* Initialize render buffers.
   * Includes steps like zeroing the buffer on the device, and optional reading of pixels from the
   * baking target. */
  bool init_render_buffers = false;

  /* Path tracing samples information. */
  struct {
    int start_sample = 0;
    int num_samples = 0;
    int sample_offset = 0;
  } path_trace;

  struct {
    /* Check for convergency and filter the mask. */
    bool filter = false;

    float threshold = 0.0f;

    /* Reset convergency flag when filtering, forcing a re-check of whether pixel did converge. */
    bool reset = false;
  } adaptive_sampling;

  struct {
    bool postprocess = false;
  } cryptomatte;

  /* Work related on the current tile. */
  struct {
    /* Write render buffers of the current tile.
     *
     * It is up to the path trace to decide whether writing should happen via user-provided
     * callback into the rendering software, or via tile manager into a partial file. */
    bool write = false;

    bool denoise = false;
  } tile;

  /* Work related on the full-frame render buffer. */
  struct {
    /* Write full render result.
     * Implies reading the partial file from disk. */
    bool write = false;
  } full;

  /* Display which is used to visualize render result. */
  struct {
    /* Display needs to be updated for the new render. */
    bool update = false;

    /* Display can use denoised result if available. */
    bool use_denoised_result = true;
  } display;

  /* Re-balance multi-device scheduling after rendering this work.
   * Note that the scheduler does not know anything about devices, so if there is only a single
   * device used, then it is up for the PathTracer to ignore the balancing. */
  bool rebalance = false;

  /* Conversion to bool, to simplify checks about whether there is anything to be done for this
   * work. */
  inline operator bool() const
  {
    return path_trace.num_samples || adaptive_sampling.filter || display.update || tile.denoise ||
           tile.write || full.write;
  }
};
```

RenderWork是一次渲染任务配置数据的集合，类似Context，包含渲染任务的配置和进展。

## 成员函数

### load_kernels

```
void PathTrace::load_kernels()
{
  if (denoiser_) {
    /* Activate graphics interop while denoiser device is created, so that it can choose a device
     * that supports interop for faster display updates. */
    if (display_ && path_trace_works_.size() > 1) {
      display_->graphics_interop_activate();
    }

    denoiser_->load_kernels(progress_);

    if (display_ && path_trace_works_.size() > 1) {
      display_->graphics_interop_deactivate();
    }
  }
}
```

加载denoiser_的kernel，算法的kernel并不是在这里加载，而是在scene中。

### alloc_work_memory

```
void PathTrace::alloc_work_memory()
{
  for (auto &&path_trace_work : path_trace_works_) {
    path_trace_work->alloc_work_memory();
  }
}
```

内存分配函数，比较直观，为其中的每一个PathTraceWork分配好内存，调用PathTraceWork自己的内存分配函数。

### reset

```
void PathTrace::reset(const BufferParams &full_params,
                      const BufferParams &big_tile_params,
                      const bool reset_rendering)
{
  if (big_tile_params_.modified(big_tile_params)) {
    big_tile_params_ = big_tile_params;
    render_state_.need_reset_params = true;
  }

  full_params_ = full_params;

  /* NOTE: GPU display checks for buffer modification and avoids unnecessary re-allocation.
   * It is requires to inform about reset whenever it happens, so that the redraw state tracking is
   * properly updated. */
  if (display_) {
    display_->reset(big_tile_params, reset_rendering);
  }

  render_state_.has_denoised_result = false;
  render_state_.tile_written = false;

  did_draw_after_reset_ = false;
}
```

重置PathTrace模块，传入的BufferParams是重新设置的参数，多次渲染时先reset再执行下一次渲染。

### render
```
void PathTrace::render(const RenderWork &render_work)
{
  /* Indicate that rendering has started and that it can be requested to cancel. */
  {
    thread_scoped_lock lock(render_cancel_.mutex);
    if (render_cancel_.is_requested) {
      return;
    }
    render_cancel_.is_rendering = true;
  }

  render_pipeline(render_work);

  /* Indicate that rendering has finished, making it so thread which requested `cancel()` can carry
   * on. */
  {
    thread_scoped_lock lock(render_cancel_.mutex);
    render_cancel_.is_rendering = false;
    render_cancel_.condition.notify_one();
  }
}
```

核心的渲染函数，调用render_pipeline执行渲染，除此之外进行渲染进程的管理，随时可以结束中断命令终止渲染。
### render_pipeline
```
void PathTrace::render_pipeline(RenderWork render_work)
{
  /* NOTE: Only check for "instant" cancel here. The user-requested cancel via progress is
   * checked in Session and the work in the event of cancel is to be finished here. */

  render_scheduler_.set_need_schedule_cryptomatte(device_scene_->data.film.cryptomatte_passes !=
                                                  0);

  render_init_kernel_execution();

  render_scheduler_.report_work_begin(render_work);

  init_render_buffers(render_work);

  rebalance(render_work);

  /* Prepare all per-thread guiding structures before we start with the next rendering
   * iteration/progression. */
  const bool use_guiding = device_scene_->data.integrator.use_guiding;
  if (use_guiding) {
    guiding_prepare_structures();
  }

  path_trace(render_work);
  if (render_cancel_.is_requested) {
    return;
  }

  /* Update the guiding field using the training data/samples collected during the rendering
   * iteration/progression. */
  const bool train_guiding = device_scene_->data.integrator.train_guiding;
  if (use_guiding && train_guiding) {
    guiding_update_structures();
  }

  adaptive_sample(render_work);
  if (render_cancel_.is_requested) {
    return;
  }

  cryptomatte_postprocess(render_work);
  if (render_cancel_.is_requested) {
    return;
  }

  denoise(render_work);
  if (render_cancel_.is_requested) {
    return;
  }

  write_tile_buffer(render_work);
  update_display(render_work);

  progress_update_if_needed(render_work);

  finalize_full_buffer_on_disk(render_work);
}
```

render_pipeline是一次渲染的全部流程，包括kernel初始化，buffer初始化，执行采样，降噪，回读RenderBuffer等。其中调用的path_trace函数执行路径追踪的采样。

### render_init_kernel_execution

```
void PathTrace::render_init_kernel_execution()
{
  for (auto &&path_trace_work : path_trace_works_) {
    path_trace_work->init_execution();
  }
}
```

运行状态的初始化和内存分配一样，调用所有PathTraceWork的初始化函数。

### init_render_buffers
```
void PathTrace::init_render_buffers(const RenderWork &render_work)
{
  update_work_buffer_params_if_needed(render_work);

  /* Handle initialization scheduled by the render scheduler. */
  if (render_work.init_render_buffers) {
    parallel_for_each(path_trace_works_, [&](unique_ptr<PathTraceWork> &path_trace_work) {
      path_trace_work->zero_render_buffers();
    });

    tile_buffer_read();
  }
}
```

初始化渲染的buffer数据，针对所有的PathTraceWork执行zero_render_buffers，将buffers清零。update_work_buffer_params_if_needed根据render_state_（当前渲染状态）判断是否需要更新buffer数据，tile_buffer_read从Device侧读取RenderBuffer，然后将其写入output_driver_中。这样上一次的渲染数据就彻底清除了。
### path_trace
```
void PathTrace::path_trace(RenderWork &render_work)
{
  if (!render_work.path_trace.num_samples) {
    return;
  }

  VLOG_WORK << "Will path trace " << render_work.path_trace.num_samples
            << " samples at the resolution divider " << render_work.resolution_divider;

  const double start_time = time_dt();

  const int num_works = path_trace_works_.size();

  thread_capture_fp_settings();

  parallel_for(0, num_works, [&](int i) {
    const double work_start_time = time_dt();
    const int num_samples = render_work.path_trace.num_samples;

    PathTraceWork *path_trace_work = path_trace_works_[i].get();
    if (path_trace_work->get_device()->have_error()) {
      return;
    }

    PathTraceWork::RenderStatistics statistics;
    path_trace_work->render_samples(statistics,
                                    render_work.path_trace.start_sample,
                                    num_samples,
                                    render_work.path_trace.sample_offset);

    const double work_time = time_dt() - work_start_time;
    work_balance_infos_[i].time_spent += work_time;
    work_balance_infos_[i].occupancy = statistics.occupancy;

    VLOG_INFO << "Rendered " << num_samples << " samples in " << work_time << " seconds ("
              << work_time / num_samples
              << " seconds per sample), occupancy: " << statistics.occupancy;
  });

  float occupancy_accum = 0.0f;
  for (const WorkBalanceInfo &balance_info : work_balance_infos_) {
    occupancy_accum += balance_info.occupancy;
  }
  const float occupancy = occupancy_accum / num_works;
  render_scheduler_.report_path_trace_occupancy(render_work, occupancy);

  render_scheduler_.report_path_trace_time(
      render_work, time_dt() - start_time, is_cancel_requested());
}
```

path_trace中调用PathTraceWork的render_samples开始采样，这里暂时忽略work_balance和性能的统计，重点在于为所有的PathTraceWork对象都调用渲染采样算法，关于PathTraceWork::render_samples在PathTraceWork中有介绍，它会根据每个kernel中path的数量在Device上执行相应的kernel算法。
### denoise
```
void PathTrace::denoise(const RenderWork &render_work)
{
  if (!render_work.tile.denoise) {
    return;
  }

  if (!denoiser_) {
    /* Denoiser was not configured, so nothing to do here. */
    return;
  }

  VLOG_WORK << "Perform denoising work.";

  const double start_time = time_dt();

  RenderBuffers *buffer_to_denoise = nullptr;
  bool allow_inplace_modification = false;

  Device *denoiser_device = denoiser_->get_denoiser_device();
  if (path_trace_works_.size() > 1 && denoiser_device && !big_tile_denoise_work_) {
    big_tile_denoise_work_ = PathTraceWork::create(denoiser_device, film_, device_scene_, nullptr);
  }

  if (big_tile_denoise_work_) {
    big_tile_denoise_work_->set_effective_buffer_params(render_state_.effective_big_tile_params,
                                                        render_state_.effective_big_tile_params,
                                                        render_state_.effective_big_tile_params);

    buffer_to_denoise = big_tile_denoise_work_->get_render_buffers();
    buffer_to_denoise->reset(render_state_.effective_big_tile_params);

    copy_to_render_buffers(buffer_to_denoise);

    allow_inplace_modification = true;
  }
  else {
    DCHECK_EQ(path_trace_works_.size(), 1);

    buffer_to_denoise = path_trace_works_.front()->get_render_buffers();
  }

  if (denoiser_->denoise_buffer(render_state_.effective_big_tile_params,
                                buffer_to_denoise,
                                get_num_samples_in_buffer(),
                                allow_inplace_modification)) {
    render_state_.has_denoised_result = true;
  }

  render_scheduler_.report_denoise_time(render_work, time_dt() - start_time);
}
```

对渲染结果进行降噪，在所有的PathTraceWork执行完毕后执行再执行。同样创建一个PathTraceWork来执行降噪算法。
### write_tile_buffer
```
void PathTrace::write_tile_buffer(const RenderWork &render_work)
{
  if (!render_work.tile.write) {
    return;
  }

  VLOG_WORK << "Write tile result.";

  render_state_.tile_written = true;

  const bool has_multiple_tiles = tile_manager_.has_multiple_tiles();

  /* Write render tile result, but only if not using tiled rendering.
   *
   * Tiles are written to a file during rendering, and written to the software at the end
   * of rendering (wither when all tiles are finished, or when rendering was requested to be
   * canceled).
   *
   * Important thing is: tile should be written to the software via callback only once. */
  if (!has_multiple_tiles) {
    VLOG_WORK << "Write tile result via buffer write callback.";
    tile_buffer_write();
  }
  /* Write tile to disk, so that the render work's render buffer can be re-used for the next tile.
   */
  else {
    VLOG_WORK << "Write tile result to disk.";
    tile_buffer_write_to_disk();
  }
}
```
写入渲染结果，根据是否有多个tiles区分，单个tile写入output_driver_，多个tiles写入disk以在下一个tile复用。