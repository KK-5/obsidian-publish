PathTraceWork可以理解为渲染任务，它会保存一次渲染所需的场景和配置信息，然后调用底层的Device算法完成渲染。PathTraceWork分为GPU和CPU两种，分别对应使用两种设备的渲染任务。同样，PathTraceWork有两个子类PathTraceWorkCPU和PathTraceWorkGPU，用于真正的实例化。

## 构造函数
PathTraceWork的构造函数如下：
```
  PathTraceWork(Device *device,
                Film *film,
                DeviceScene *device_scene,
                bool *cancel_requested_flag)
```
他是一个保护的成员函数，这也就意味着PathTraceWork是不能随意实例化的。PathTraceWork的实例化使用提供的Create函数：
```
static unique_ptr<PathTraceWork> create(Device *device,
                                          Film *film,
                                          DeviceScene *device_scene,
                                          bool *cancel_requested_flag)
```
一个静态的成员函数，提供PathTraceWork的构造方法。它的实现如下：
```
unique_ptr<PathTraceWork> PathTraceWork::create(Device *device,
                                                Film *film,
                                                DeviceScene *device_scene,
                                                bool *cancel_requested_flag)
{
  if (device->info.type == DEVICE_CPU) {
    return make_unique<PathTraceWorkCPU>(device, film, device_scene, cancel_requested_flag);
  }
  if (device->info.type == DEVICE_DUMMY) {
    /* Dummy devices can't perform any work. */
    return nullptr;
  }

  return make_unique<PathTraceWorkGPU>(device, film, device_scene, cancel_requested_flag);
}
```
明显可以看到create根据不同的Device类型来实例化不同的PathTraceWork类，真正调用的使子类的构造函数。

## 成员数据

```
  /* Device which will be used for path tracing.
   * Note that it is an actual render device (and never is a multi-device). */
  Device *device_;

  /* Film is used to access display pass configuration for GPU display update.
   * Note that only fields which are not a part of kernel data can be accessed via the Film. */
  Film *film_;

  /* Device side scene storage, that may be used for integrator logic. */
  DeviceScene *device_scene_;

  /* Render buffers where sampling is being accumulated into, allocated for a fraction of the big
   * tile which is being rendered by this work.
   * It also defines possible subset of a big tile in the case of multi-device rendering. */
  unique_ptr<RenderBuffers> buffers_;

  /* Effective parameters of the full, big tile, and current work render buffer.
   * The latter might be different from `buffers_->params` when there is a resolution divider
   * involved. */
  BufferParams effective_full_params_;
  BufferParams effective_big_tile_params_;
  BufferParams effective_buffer_params_;
```

通过注释可以知道这些成员的含义。device_是渲染设备的引用，film_用于显示更新，device_scene_是场景，buffers_是渲染缓冲区，也就是保存渲染结果的地方。下面的三个BufferParams是渲染的buffer参数，effective_full_params_表示全部buffer的大小，目前没有找到使用的地方（可能是后续扩展）。effective_big_tile_params_表示整个图片的buffer（一个图片被分成了多个tile，也有可能只有一个tile）。effective_buffer_params_表示当前渲染的tile的buffer，是effective_big_tile_params_的一部分。

## 成员函数

PathTraceWork最核心的成员函数是render_samples，这个函数表示开始进行采样渲染图片，它在CPU和GPU的实现不一样。除此之外，PathTraceWork还提供了内存访问函数copy_to_render_buffers和copy_from_render_buffers，这两个函数风别可以向buffers_中写入和读取Device的数据。

以copy_to_render_buffers为例：

```
void PathTraceWork::copy_to_render_buffers(RenderBuffers *render_buffers)
{
  // 从device复制数据，由子类PathTraceWorkCPU或PathTraceWorkGPU实现
  copy_render_buffers_from_device();

  // 通过effective_buffer_params_计算内存大小
  const int64_t width = effective_buffer_params_.width;
  const int64_t height = effective_buffer_params_.height;
  const int64_t pass_stride = effective_buffer_params_.pass_stride;
  const int64_t row_stride = width * pass_stride;
  const int64_t data_size = row_stride * height * sizeof(float);

  // 计算偏移， offset = offset_y * row_stride
  const int64_t offset_y = effective_buffer_params_.full_y - effective_big_tile_params_.full_y;
  const int64_t offset_in_floats = offset_y * row_stride;

  const float *src = buffers_->buffer.data();
  float *dst = render_buffers->buffer.data() + offset_in_floats;
  // memcpy复制数据
  memcpy(dst, src, data_size);
}
```

## PathTraceWorkGPU

PathTraceWorkGPU是PathTraceWork的子类，表示运行在GPU上的渲染任务，它的具体实现很复杂，除了PathTraceWork的基本功能外，还包括kernel算法的管理，采样路径的管理等。

PathTraceWorkGPU中重要的成员函数：

### init_execution

```
void PathTraceWorkGPU::init_execution()
{
  // DeviceQueue的初始化
  queue_->init_execution();

  // 积分器状态拷贝
  /* Copy to device side struct in constant memory. */
  device_->const_copy_to(
      "integrator_state", &integrator_state_gpu_, sizeof(integrator_state_gpu_));
}
```

主要用于运行状态的初始化。

### alloc_work_memory

```
void PathTraceWorkGPU::alloc_work_memory()
{
  // soa = structure of arrays;为integrator结构体分配内存
  alloc_integrator_soa();
  
  // 为integrator_queue_counter_, num_queued_paths_, queued_paths_分配内存
  // integrator_queue_counter_是此次渲染使用的kernel算法缓存,哈希表，保存了每个kernel的路径数量
  // num_queued_paths_,queued_paths_为临时数据，保存特定内核的path队列
  alloc_integrator_queue();
  
  // 分配排序分区的内存，排序分区用于提升性能，这里暂时忽略
  alloc_integrator_sorting();
  
  // 分配integrator_next_shadow_path_index_和integrator_next_main_path_index_的内存
  // 用于路径分离（?）
  alloc_integrator_path_split();
}
```

为当前渲染任务分配必须的内存。

### render_samples

```
void PathTraceWorkGPU::render_samples(RenderStatistics &statistics,
                                      int start_sample,
                                      int samples_num,
                                      int sample_offset)
{
  /* Limit number of states for the tile and rely on a greedy scheduling of tiles. This allows to
   * add more work (because tiles are smaller, so there is higher chance that more paths will
   * become busy after adding new tiles). This is especially important for the shadow catcher which
   * schedules work in halves of available number of paths. */
  // 调度算法的优化
  work_tile_scheduler_.set_max_num_path_states(max_num_paths_ / 8);
  work_tile_scheduler_.set_accelerated_rt((device_->get_bvh_layout_mask() & BVH_LAYOUT_OPTIX) !=
                                          0);
  work_tile_scheduler_.reset(effective_buffer_params_,
                             start_sample,
                             samples_num,
                             sample_offset,
                             device_scene_->data.integrator.scrambling_distance);
  // 重置DeviceQueue
  enqueue_reset();

  int num_iterations = 0;
  uint64_t num_busy_accum = 0;

  /* TODO: set a hard limit in case of undetected kernel failures? */
  while (true) {
    /* Enqueue work from the scheduler, on start or when there are not enough
     * paths to keep the device occupied. */
    bool finished;
    // enqueue_work_tiles调用Device侧算法
    if (enqueue_work_tiles(finished)) {
      /* Copy stats from the device. */
      queue_->copy_from_device(integrator_queue_counter_);

      if (!queue_->synchronize()) {
        break; /* Stop on error. */
      }
    }

    if (is_cancel_requested()) {
      break;
    }

    /* Stop if no more work remaining. */
    if (finished) {
      break;
    }

    /* Enqueue on of the path iteration kernels. */
    // enqueue_path_iteration也调用Device侧算法
    if (enqueue_path_iteration()) {
      /* Copy stats from the device. */
      queue_->copy_from_device(integrator_queue_counter_);

      if (!queue_->synchronize()) {
        break; /* Stop on error. */
      }
    }

    if (is_cancel_requested()) {
      break;
    }

    num_busy_accum += num_active_main_paths_paths();
    ++num_iterations;
  }

  statistics.occupancy = static_cast<float>(num_busy_accum) / num_iterations / max_num_paths_;
}
```

渲染的核心函数，循环使用enqueue_work_tiles执行渲染算法，并且同步渲染数据，如果没有finished，继续执行enqueue_path_iteration。

### enqueue_work_tiles

```
bool PathTraceWorkGPU::enqueue_work_tiles(bool &finished)
{
  /* If there are existing paths wait them to go to intersect closest kernel, which will align the
   * wavefront of the existing and newly added paths. */
  /* TODO: Check whether counting new intersection kernels here will have positive affect on the
   * performance. */
  // get_most_queued_kernel获取所有kernel中路径数量最多的一个（算法是通过path的数量来决定执行顺序的）
  const DeviceKernel kernel = get_most_queued_kernel();
  if (kernel != DEVICE_KERNEL_NUM && kernel != DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST) {
    return false;
  }
  // 获取所有kernel的路径总数，shadow_path除外
  int num_active_paths = num_active_main_paths_paths();

  /* Don't schedule more work if canceling. */
  if (is_cancel_requested()) {
    if (num_active_paths == 0) {
      finished = true;
    }
    return false;
  }

  finished = false;
  // 划分tiles
  vector<KernelWorkTile> work_tiles;

  int max_num_camera_paths = max_num_paths_;
  int num_predicted_splits = 0;

  if (has_shadow_catcher()) {
    /* When there are shadow catchers in the scene bounce from them will split the state. So we
     * make sure there is enough space in the path states array to fit split states.
     *
     * Basically, when adding N new paths we ensure that there is 2*N available path states, so
     * that all the new paths can be split.
     *
     * Note that it is possible that some of the current states can still split, so need to make
     * sure there is enough space for them as well. */

    /* Number of currently in-flight states which can still split. */
    const int num_scheduled_possible_split = shadow_catcher_count_possible_splits();

    const int num_available_paths = max_num_paths_ - num_active_paths;
    const int num_new_paths = num_available_paths / 2;
    max_num_camera_paths = max(num_active_paths,
                               num_active_paths + num_new_paths - num_scheduled_possible_split);
    num_predicted_splits += num_scheduled_possible_split + num_new_paths;
  }

  /* Schedule when we're out of paths or there are too few paths to keep the
   * device occupied. */
  int num_paths = num_active_paths;
  if (num_paths == 0 || num_paths < min_num_active_main_paths_) {
    /* Get work tiles until the maximum number of path is reached. */
    while (num_paths < max_num_camera_paths) {
      KernelWorkTile work_tile;
      // 使用work_tile_scheduler_构建tile，直到路径数量达到max_num_camera_paths
      if (work_tile_scheduler_.get_work(&work_tile, max_num_camera_paths - num_paths)) {
        work_tiles.push_back(work_tile);
        // 路径数量 = 像素的数量 * 每个像素的采样数
        num_paths += work_tile.w * work_tile.h * work_tile.num_samples;
      }
      else {
        break;
      }
    }

    /* If we couldn't get any more tiles, we're done. */
    if (work_tiles.size() == 0 && num_paths == 0) {
      finished = true;
      return false;
    }
  }

  /* Initialize paths from work tiles. */
  if (work_tiles.size() == 0) {
    return false;
  }

  /* Compact state array when number of paths becomes small relative to the
   * known maximum path index, which makes computing active index arrays slow. */
  compact_main_paths(num_active_paths);

  if (has_shadow_catcher()) {
    integrator_next_main_path_index_.data()[0] = num_paths;
    queue_->copy_to_device(integrator_next_main_path_index_);
  }

  enqueue_work_tiles((device_scene_->data.bake.use) ? DEVICE_KERNEL_INTEGRATOR_INIT_FROM_BAKE :
                                                      DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA,
                     work_tiles.data(),
                     work_tiles.size(),
                     num_active_paths,
                     num_predicted_splits);

  return true;
}
```

主要做渲染时路径数量的调整，生成tile，和一些性能优化。最终又会调用enqueue_work_tiles这个同名函数，这时才真正执行kernel算法。

注意在函数最开始时，如果获取到的kernel既不是DEVICE_KERNEL_NUM也不是DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST，此函数就会返回，DEVICE_KERNEL_NUM就表示并没有获取到kernel（所有kernel中的路径数量都是0，初始状态就是这样），另一种情况就是DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST中路径数量最多，需要最优先执行时，这个函数才会运行。而这个函数最终执行的kernel是DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA或DEVICE_KERNEL_INTEGRATOR_INIT_FROM_BAKE。猜想这应该渲染是最开始和DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST前执行的函数，后面就不执行了。

在调用DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA时，这里提供了4个参数，在理解这4个参数的含义之前，先看一下work_tile的定义：

```
/* Work Tiles */
typedef struct KernelWorkTile {
  uint x, y, w, h;   // tile的位置，x, y是起点，w, h是宽高

  // start_sample, sample_offset用来表示采样偏移，cycles引擎支持这种特性，暂不清楚它的功能
  // 先忽略
  uint start_sample;  
  uint num_samples;   // 采样数
  uint sample_offset;
  
  // 用来定位tile对应的render_buffer的位置?
  int offset;
  uint stride;

  /* Precalculated parameters used by init_from_camera kernel on GPU. */
  int path_index_offset;
  int work_size;
} KernelWorkTile;
```

再看4个参数：

- work_tiles.data()：此次渲染分解出的所有tile
    
- work_tiles.size()：此次渲染分解出的tile的数量
    
- num_active_paths：激活的路径数量，计算了队列中所有kernel的path的总和（为什么这么计算）
    
- num_predicted_splits：先忽略，暂时任务它是0
    

上面的tile的计算依赖于max_num_paths_，这是设备可以承受的最大path数量，也就是设备的最大线程数，所以这里是根据具体设备来分解tiles。

### enqueue_work_tiles(…)

```
void PathTraceWorkGPU::enqueue_work_tiles(DeviceKernel kernel,
                                          const KernelWorkTile work_tiles[],
                                          const int num_work_tiles,
                                          const int num_active_paths,
                                          const int num_predicted_splits)
{
  // 这个函数的DeviceKernel只能是DEVICE_KERNEL_INTEGRATOR_INIT_FROM_BAKE或者
  // DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA
  // work_tiles_是会实时调整的。。
  /* Copy work tiles to device. */
  if (work_tiles_.size() < num_work_tiles) {
    work_tiles_.alloc(num_work_tiles);
  }

  int path_index_offset = num_active_paths;
  int max_tile_work_size = 0;
  // 计算max_tile_work_size和path_index_offset，即最多的tile采样数量和path索引的偏移
  for (int i = 0; i < num_work_tiles; i++) {
    // 这里是把传入的work_tiles里面的数据复制到了work_tiles_里面
    KernelWorkTile &work_tile = work_tiles_.data()[i];
    work_tile = work_tiles[i];

    const int tile_work_size = work_tile.w * work_tile.h * work_tile.num_samples;

    work_tile.path_index_offset = path_index_offset;
    // work_size在这里初始化，也就是一个work_tile中的采样数量，按照描述，它只用与camera init
    work_tile.work_size = tile_work_size;

    path_index_offset += tile_work_size;

    max_tile_work_size = max(max_tile_work_size, tile_work_size);
  }
  // 拷贝到Device
  queue_->copy_to_device(work_tiles_);

  device_ptr d_work_tiles = work_tiles_.device_pointer;
  device_ptr d_render_buffer = buffers_->buffer.device_pointer;

  /* Launch kernel. */
  DeviceKernelArguments args(
      &d_work_tiles, &num_work_tiles, &d_render_buffer, &max_tile_work_size);
      
  // 执行kernel算法
  // max_tile_work_size * num_work_tiles 表示所有work_tile的size都是视作最大的size，
  // 这个参数用于device上的线程数分配
  queue_->enqueue(kernel, max_tile_work_size * num_work_tiles, args);
  // max_active_main_path_index_意为最大的active路径的索引，所有路径索引都不应该超出这个值
  max_active_main_path_index_ = path_index_offset + num_predicted_splits;
}
```

这个函数可理解为整个渲染算法执行的第一步，之前划分好每一个tile之后，在这里进行渲染场景的初始化，分为Init_From_Camera和Init_From_Bake两种，调用不同的kernel算法来实现。

### get_most_queued_kernel

```
DeviceKernel PathTraceWorkGPU::get_most_queued_kernel() const
{
  const IntegratorQueueCounter *queue_counter = integrator_queue_counter_.data();

  int max_num_queued = 0;
  DeviceKernel kernel = DEVICE_KERNEL_NUM;

  for (int i = 0; i < DEVICE_KERNEL_INTEGRATOR_NUM; i++) {
    if (queue_counter->num_queued[i] > max_num_queued) {
      kernel = (DeviceKernel)i;
      max_num_queued = queue_counter->num_queued[i];
    }
  }

  return kernel;
}
```

获得所有需执行kernel（保存在integrator_queue_counter_中path数量最多的一个kernel）。在enqueue_work_tiles中使用。

### num_active_main_paths_paths

```
int PathTraceWorkGPU::num_active_main_paths_paths()
{
  IntegratorQueueCounter *queue_counter = integrator_queue_counter_.data();

  int num_paths = 0;
  for (int i = 0; i < DEVICE_KERNEL_INTEGRATOR_NUM; i++) {
    DCHECK_GE(queue_counter->num_queued[i], 0)
        << "Invalid number of queued states for kernel "
        << device_kernel_as_string(static_cast<DeviceKernel>(i));

    if (!kernel_is_shadow_path((DeviceKernel)i)) {
      num_paths += queue_counter->num_queued[i];
    }
  }

  return num_paths;
}
```

计算所有需执行的kernel的path数量总和。

### enqueue_path_iteration

```
bool PathTraceWorkGPU::enqueue_path_iteration()
{
  /* Find kernel to execute, with max number of queued paths. */
  const IntegratorQueueCounter *queue_counter = integrator_queue_counter_.data();
  // 计算path总数
  int num_active_paths = 0;
  for (int i = 0; i < DEVICE_KERNEL_INTEGRATOR_NUM; i++) {
    num_active_paths += queue_counter->num_queued[i];
  }

  if (num_active_paths == 0) {
    return false;
  }

  /* Find kernel to execute, with max number of queued paths. */
  const DeviceKernel kernel = get_most_queued_kernel();
  if (kernel == DEVICE_KERNEL_NUM) {
    return false;
  }

  /* For kernels that add shadow paths, check if there is enough space available.
   * If not, schedule shadow kernels first to clear out the shadow paths. */
  int num_paths_limit = INT_MAX;
  if (kernel_creates_shadow_paths(kernel)) {
    compact_shadow_paths();

    const int available_shadow_paths = max_num_paths_ -
                                       integrator_next_shadow_path_index_.data()[0];
    if (available_shadow_paths < queue_counter->num_queued[kernel]) {
      if (queue_counter->num_queued[DEVICE_KERNEL_INTEGRATOR_INTERSECT_SHADOW]) {
        enqueue_path_iteration(DEVICE_KERNEL_INTEGRATOR_INTERSECT_SHADOW);
        return true;
      }
      else if (queue_counter->num_queued[DEVICE_KERNEL_INTEGRATOR_SHADE_SHADOW]) {
        enqueue_path_iteration(DEVICE_KERNEL_INTEGRATOR_SHADE_SHADOW);
        return true;
      }
    }
    else if (kernel_creates_ao_paths(kernel)) {
      /* AO kernel creates two shadow paths, so limit number of states to schedule. */
      num_paths_limit = available_shadow_paths / 2;
    }
  }

  /* Schedule kernel with maximum number of queued items. */
  enqueue_path_iteration(kernel, num_paths_limit);

  /* Update next shadow path index for kernels that can add shadow paths. */
  if (kernel_creates_shadow_paths(kernel)) {
    queue_->copy_from_device(integrator_next_shadow_path_index_);
  }

  return true;
}
```

每一次都将path数最多的kernel加入队列执行，其中阴影path的执行需要考虑到内存是否充足。重点是调整

num_paths_limit，即路径的数量限制。

### enqueue_path_iteration(…)

```
void PathTraceWorkGPU::enqueue_path_iteration(DeviceKernel kernel, const int num_paths_limit)
{
  device_ptr d_path_index = 0;

  /* Create array of path indices for which this kernel is queued to be executed. */
  // work_size = max_active_main_path_index_，即最大的路径索引，在enqueue_work_tiles中更新过它的值
  int work_size = kernel_max_active_main_path_index(kernel);

  IntegratorQueueCounter *queue_counter = integrator_queue_counter_.data();
  int num_queued = queue_counter->num_queued[kernel];
  // compute_queued_paths这个过程应该是可以计算某个kernel激活的所有路径，暂不清楚算法
  if (kernel_uses_sorting(kernel)) {
    /* Compute array of active paths, sorted by shader. */
    work_size = num_queued;
    d_path_index = queued_paths_.device_pointer;

    compute_sorted_queued_paths(kernel, num_paths_limit);
  }
  else if (num_queued < work_size) {
    work_size = num_queued;
    d_path_index = queued_paths_.device_pointer;

    if (kernel_is_shadow_path(kernel)) {
      /* Compute array of active shadow paths for specific kernel. */
      compute_queued_paths(DEVICE_KERNEL_INTEGRATOR_QUEUED_SHADOW_PATHS_ARRAY, kernel);
    }
    else {
      /* Compute array of active paths for specific kernel. */
      compute_queued_paths(DEVICE_KERNEL_INTEGRATOR_QUEUED_PATHS_ARRAY, kernel);
    }
  }

  work_size = min(work_size, num_paths_limit);

  DCHECK_LE(work_size, max_num_paths_);
  // 根据不同的kernel类型填充参数，DeviceQueue执行kernel
  switch (kernel) {
    case DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST: {
      /* Closest ray intersection kernels with integrator state and render buffer. */
      DeviceKernelArguments args(&d_path_index, &buffers_->buffer.device_pointer, &work_size);

      queue_->enqueue(kernel, work_size, args);
      break;
    }

    case DEVICE_KERNEL_INTEGRATOR_INTERSECT_SHADOW:
    case DEVICE_KERNEL_INTEGRATOR_INTERSECT_SUBSURFACE:
    case DEVICE_KERNEL_INTEGRATOR_INTERSECT_VOLUME_STACK: {
      /* Ray intersection kernels with integrator state. */
      DeviceKernelArguments args(&d_path_index, &work_size);

      queue_->enqueue(kernel, work_size, args);
      break;
    }
    case DEVICE_KERNEL_INTEGRATOR_SHADE_BACKGROUND:
    case DEVICE_KERNEL_INTEGRATOR_SHADE_LIGHT:
    case DEVICE_KERNEL_INTEGRATOR_SHADE_SHADOW:
    case DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE:
    case DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_RAYTRACE:
    case DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_MNEE:
    case DEVICE_KERNEL_INTEGRATOR_SHADE_VOLUME: {
      /* Shading kernels with integrator state and render buffer. */
      DeviceKernelArguments args(&d_path_index, &buffers_->buffer.device_pointer, &work_size);

      queue_->enqueue(kernel, work_size, args);
      break;
    }

    default:
      LOG(FATAL) << "Unhandled kernel " << device_kernel_as_string(kernel)
                 << " used for path iteration, should never happen.";
      break;
  }
}
```

这个方法可以看作一个迭代的算法，每一次执行路径数量最多的kernel，直到将kernel执行完毕。当将一个kernel取出执行后，它的路径数量应该会减少，这个过程并不是在这里完成，实在GPU执行kernel的时候完成，因为在render_samples函数中会将GPU的integrator_queue_counter_复制回来。

### copy_render_buffers_from_device

```
bool PathTraceWorkGPU::copy_render_buffers_from_device()
{
  queue_->copy_from_device(buffers_->buffer);

  /* Synchronize so that the CPU-side buffer is available at the exit of this function. */
  return queue_->synchronize();
}
```

PathTraceWorkGPU的内存管理函数，从Device中获取RenderBuffer，在父类的copy_render_buffers_from_device中调用。同样，有一个copy_render_buffers_to_device向Device中拷入数据。