Session是渲染器的应用层，用来运行渲染循环和发布渲染任务，也可以理解成渲染器的接口，再上层的app就是对Session的封装和调用以提供用户使用环境。

## 关联的数据结构

### SceneParams

```
class SceneParams {
 public:
  ShadingSystem shadingsystem;

  /* Requested BVH layout.
   *
   * If it's not supported by the device, the widest one from supported ones
   * will be used, but BVH wider than this one will never be used.
   */
  BVHLayout bvh_layout;

  BVHType bvh_type;
  bool use_bvh_spatial_split;
  bool use_bvh_compact_structure;
  bool use_bvh_unaligned_nodes;
  int num_bvh_time_steps;
  int hair_subdivisions;
  CurveShapeType hair_shape;
  int texture_limit;

  bool background;
  
  ...
}
```
场景的配置参数，包括Shading系统，bvh配置，hair参数等。
### SessionParams

```
class SessionParams {
 public:
  DeviceInfo device;

  bool headless;
  bool background;

  bool experimental;
  int samples;
  int sample_offset;
  int pixel_size;
  int threads;

  /* Limit in seconds for how long path tracing is allowed to happen.
   * Zero means no limit is applied. */
  double time_limit;

  bool use_profiling;

  bool use_auto_tile;
  int tile_size;

  bool use_resolution_divider;

  ShadingSystem shadingsystem;

  /* Session-specific temporary directory to store in-progress EXR files in. */
  string temp_dir;
  ...
}
```

Session的配置参数，包括设备信息，渲染的采样信息，tile配置，Shading系统等。

## 构造函数

```
explicit Session(const SessionParams &params_, const SceneParams &scene_params)
    : params(params_), render_scheduler_(tile_manager_, params)
{
  TaskScheduler::init(params.threads);

  delayed_reset_.do_reset = false;

  pause_ = false;
  new_work_added_ = false;

  device = Device::create(params.device, stats, profiler);

  if (device->have_error()) {
    progress.set_error(device->error_message());
  }

  scene = new Scene(scene_params, device);

  /* Configure path tracer. */
  path_trace_ = make_unique<PathTrace>(
      device, scene->film, &scene->dscene, render_scheduler_, tile_manager_);
  path_trace_->set_progress(&progress);
  path_trace_->progress_update_cb = [&]() { update_status_time(); };

  tile_manager_.full_buffer_written_cb = [&](string_view filename) {
    if (!full_buffer_written_cb) {
      return;
    }
    full_buffer_written_cb(filename);
  };

  /* Create session thread. */
  session_thread_ = new thread(function_bind(&Session::thread_run, this));
}
```

Session的构造函数，包括TaskScheduler初始化，Device创建，Scene创建，PathTrace实例化，tile_manager_回调函数设置。并且Session创建时自动创建一个线程并运行thread_run函数。

## 成员函数

### start
```
void Session::start()
{
  {
    /* Signal session thread to start rendering. */
    thread_scoped_lock session_thread_lock(session_thread_mutex_);
    if (session_thread_state_ == SESSION_THREAD_RENDER) {
      /* Already rendering, nothing to do. */
      return;
    }
    session_thread_state_ = SESSION_THREAD_RENDER;
  }

  session_thread_cond_.notify_all();
}
```
此session开始运行，session_thread_lock用来保证session_thread_state_ 的原子性，session_thread_cond_通知子线程解除阻塞。
### cancel
```
void Session::cancel(bool quick)
{
  /* Cancel any long running device operations (e.g. shader compilations). */
  device->cancel();

  /* Check if session thread is rendering. */
  const bool rendering = is_session_thread_rendering();

  if (rendering) {
    /* Cancel path trace operations. */
    if (quick && path_trace_) {
      path_trace_->cancel();
    }

    /* Cancel other operations. */
    progress.set_cancel("Exiting");

    /* Signal unpause in case the render was paused. */
    {
      thread_scoped_lock pause_lock(pause_mutex_);
      pause_ = false;
    }
    pause_cond_.notify_all();

    /* Wait for render thread to be cancelled or finished. */
    wait();
  }
}
```
session终止运行，检擦子线程运行状态，如果正在运行pause_cond_通知进入暂停，然后进入wait状态等待渲染线程全部退出。
### wait
```
void Session::wait()
{
  /* Wait until session thread either is waiting or ending. */
  while (true) {
    thread_scoped_lock session_thread_lock(session_thread_mutex_);
    if (session_thread_state_ != SESSION_THREAD_RENDER) {
      break;
    }
    session_thread_cond_.wait(session_thread_lock);
  }
}
```
进入死循环，直到session_thread_state_ 变为非RENDER状态，但前提是收到session_thread_cond_的通知，它会在start中发出。

### set_pause
```
void Session::set_pause(bool pause)
{
  bool notify = false;

  {
    thread_scoped_lock pause_lock(pause_mutex_);

    if (pause != pause_) {
      pause_ = pause;
      notify = true;
    }
  }

  if (is_session_thread_rendering()) {
    if (notify) {
      pause_cond_.notify_all();
    }
  }
  else if (pause_) {
    update_status_time(pause_);
  }
}
```

session暂停，修改pause_ 状态，pause_cond_发送通知，和cancel时有些相似，但是不对让渲染线程退出。

### thread_run
```
void Session::thread_run()
{
  while (true) {
    {
      thread_scoped_lock session_thread_lock(session_thread_mutex_);

      if (session_thread_state_ == SESSION_THREAD_WAIT) {
        /* Continue waiting for any signal from the main thread. */
        session_thread_cond_.wait(session_thread_lock);
        continue;
      }
      else if (session_thread_state_ == SESSION_THREAD_END) {
        /* End thread immediately. */
        break;
      }
    }

    /* Execute a render. */
    thread_render();

    /* Go back from rendering to waiting. */
    {
      thread_scoped_lock session_thread_lock(session_thread_mutex_);
      if (session_thread_state_ == SESSION_THREAD_RENDER) {
        session_thread_state_ = SESSION_THREAD_WAIT;
      }
    }
    session_thread_cond_.notify_all();
  }

  /* Flush any remaining operations and destroy display driver here. This ensure
   * graphics API resources are created and destroyed all in the session thread,
   * which can avoid problems contexts and multiple threads. */
  path_trace_->flush_display();
  path_trace_->set_display_driver(nullptr);
}
```

渲染线程，在Session创建的时候就会创建，初始是SESSION_THREAD_WAIT状态，当start()开始调用时，session_thread_cond_收到通知，阻塞解除，进入thread_render函数进行渲染，然后又将状态置为SESSION_THREAD_WAIT。

### thread_render

```
void Session::thread_render()
{
  if (params.use_profiling && (params.device.type == DEVICE_CPU)) {
    profiler.start();
  }

  /* session thread loop */
  progress.set_status("Waiting for render to start");

  /* run */
  if (!progress.get_cancel()) {
    /* reset number of rendered samples */
    progress.reset_sample();

    run_main_render_loop();
  }

  profiler.stop();

  /* progress update */
  if (progress.get_cancel())
    progress.set_status(progress.get_cancel_message());
  else
    progress.set_update();
}
```
渲染函数，progress管理渲染进程，run_main_render_loop是渲染主循环。

### run_main_render_loop

```
void Session::run_main_render_loop()
{
  path_trace_->clear_display();

  while (true) {
    RenderWork render_work = run_update_for_next_iteration();

    if (!render_work) {
      if (VLOG_INFO_IS_ON) {
        double total_time, render_time;
        progress.get_time(total_time, render_time);
        VLOG_INFO << "Rendering in main loop is done in " << render_time << " seconds.";
        VLOG_INFO << path_trace_->full_report();
      }

      if (params.background) {
        /* if no work left and in background mode, we can stop immediately. */
        progress.set_status("Finished");
        break;
      }
    }

    const bool did_cancel = progress.get_cancel();
    if (did_cancel) {
      render_scheduler_.render_work_reschedule_on_cancel(render_work);
      if (!render_work) {
        break;
      }
    }
    else if (run_wait_for_work(render_work)) {
      continue;
    }

    /* Stop rendering if error happened during scene update or other step of preparing scene
     * for render. */
    if (device->have_error()) {
      progress.set_error(device->error_message());
      break;
    }

    {
      /* buffers mutex is locked entirely while rendering each
       * sample, and released/reacquired on each iteration to allow
       * reset and draw in between */
      thread_scoped_lock buffers_lock(buffers_mutex_);

      /* update status and timing */
      update_status_time();

      /* render */
      path_trace_->render(render_work);

      /* update status and timing */
      update_status_time();

      /* Stop rendering if error happened during path tracing. */
      if (device->have_error()) {
        progress.set_error(device->error_message());
        break;
      }
    }

    progress.set_update();

    if (did_cancel) {
      break;
    }
  }
}
```

渲染主要循环，run_update_for_next_iteration获取RenderWork，buffers_lock用来保证多线程中buffer数据的正确性。path_trace_->render(render_work)调用了PathWork模块的render函数，也就是执行路径追踪渲染算法的函数。在渲染之前，还有一个步骤是run_wait_for_work，在这里线程会一直阻塞直到session暂停解除（如果当前session是暂停的）或者新的RenderWork被加入进来。

### run_update_for_next_iteration

```
RenderWork Session::run_update_for_next_iteration()
{
  RenderWork render_work;

  thread_scoped_lock scene_lock(scene->mutex);

  bool have_tiles = true;
  bool switched_to_new_tile = false;
  bool did_reset = false;

  /* Perform delayed reset if requested. */
  {
    thread_scoped_lock reset_lock(delayed_reset_.mutex);
    if (delayed_reset_.do_reset) {
      did_reset = true;

      thread_scoped_lock buffers_lock(buffers_mutex_);
      do_delayed_reset();

      /* After reset make sure the tile manager is at the first big tile. */
      have_tiles = tile_manager_.next();
      switched_to_new_tile = true;
    }
  }

  /* Update number of samples in the integrator.
   * Ideally this would need to happen once in `Session::set_samples()`, but the issue there is
   * the initial configuration when Session is created where the `set_samples()` is not used.
   *
   * NOTE: Unless reset was requested only allow increasing number of samples. */
  if (did_reset || scene->integrator->get_aa_samples() < params.samples) {
    scene->integrator->set_aa_samples(params.samples);
  }

  /* Update denoiser settings. */
  {
    const DenoiseParams denoise_params = scene->integrator->get_denoise_params();
    path_trace_->set_denoiser_params(denoise_params);
  }

  /* Update adaptive sampling. */
  {
    const AdaptiveSampling adaptive_sampling = scene->integrator->get_adaptive_sampling();
    path_trace_->set_adaptive_sampling(adaptive_sampling);
  }

  /* Update path guiding. */
  {
    const GuidingParams guiding_params = scene->integrator->get_guiding_params(device);
    const bool guiding_reset = (guiding_params.use) ? scene->need_reset(false) : false;
    path_trace_->set_guiding_params(guiding_params, guiding_reset);
  }

  render_scheduler_.set_num_samples(params.samples);
  render_scheduler_.set_start_sample(params.sample_offset);
  render_scheduler_.set_time_limit(params.time_limit);

  while (have_tiles) {
    render_work = render_scheduler_.get_render_work();
    if (render_work) {
      break;
    }

    progress.add_finished_tile(false);

    have_tiles = tile_manager_.next();
    if (have_tiles) {
      render_scheduler_.reset_for_next_tile();
      switched_to_new_tile = true;
    }
  }

  if (render_work) {
    scoped_timer update_timer;

    if (switched_to_new_tile) {
      BufferParams tile_params = buffer_params_;

      const Tile &tile = tile_manager_.get_current_tile();

      tile_params.width = tile.width;
      tile_params.height = tile.height;

      tile_params.window_x = tile.window_x;
      tile_params.window_y = tile.window_y;
      tile_params.window_width = tile.window_width;
      tile_params.window_height = tile.window_height;

      tile_params.full_x = tile.x + buffer_params_.full_x;
      tile_params.full_y = tile.y + buffer_params_.full_y;
      tile_params.full_width = buffer_params_.full_width;
      tile_params.full_height = buffer_params_.full_height;

      tile_params.update_offset_stride();

      path_trace_->reset(buffer_params_, tile_params, did_reset);
    }

    const int resolution = render_work.resolution_divider;
    const int width = max(1, buffer_params_.full_width / resolution);
    const int height = max(1, buffer_params_.full_height / resolution);

    {
      /* Load render kernels, before device update where we upload data to the GPU.
       * Do it outside of the scene mutex since the heavy part of the loading (i.e. kernel
       * compilation) does not depend on the scene and some other functionality (like display
       * driver) might be waiting on the scene mutex to synchronize display pass.
       *
       * The scene will lock itself for the short period if it needs to update kernel features. */
      scene_lock.unlock();
      scene->load_kernels(progress);
      scene_lock.lock();
    }

    if (update_scene(width, height)) {
      profiler.reset(scene->shaders.size(), scene->objects.size());
    }

    /* Unlock scene mutex before loading denoiser kernels, since that may attempt to activate
     * graphics interop, which can deadlock when the scene mutex is still being held. */
    scene_lock.unlock();

    path_trace_->load_kernels();
    path_trace_->alloc_work_memory();

    /* Wait for device to be ready (e.g. finish any background compilations). */
    string device_status;
    while (!device->is_ready(device_status)) {
      progress.set_status(device_status);
      if (progress.get_cancel()) {
        break;
      }
      std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }

    progress.add_skip_time(update_timer, params.background);
  }

  return render_work;
}
```

run_update_for_next_iteration可理解为渲染前的设置和准备工作，在这里将渲染参数设置到path_trace_中，同时读取后续的tile（如果有的话）,然后使用新的tile设置reset path_trace_。同时，在这里使用 scene->load_kernels 加载场景中需要的kernel算法。这里也从render_scheduler_中获得RenderWork返回给run_main_render_loop中使用。