## DeviceQueue

DeviceQueue类似于DX12中的CommandQueue，它提供调用Device算法的接口，是Host于Device交互的桥梁。

DeviceQueue属于Cycles的Device模块，该模块是针对不同渲染设备的抽象，CUDADeviceQueue就是属于DeviceQueue的一种。

DeviceQueue类是一个抽象类，具体的实现由每种不同的Device决定。

### DeviceKernelArguments

这是一个用于保证设备内核参数与类型正确性的容器，他作为enqueue函数的参数，因为不同Device的DeviceQueue所需要的参数类型和类型都不一样，使用DeviceKernelArguments将他们统一。

实现：

```
struct DeviceKernelArguments {
  enum Type {
    POINTER,
    INT32,
    FLOAT32,
    BOOLEAN,
    KERNEL_FILM_CONVERT,
  };
  // 最多有18个参数
  static const int MAX_ARGS = 18;
  // 保存每个参数的类型、值、所占内存大小
  Type types[MAX_ARGS];
  void *values[MAX_ARGS];
  size_t sizes[MAX_ARGS];
  // 已add的参数个数
  size_t count = 0;
  
  // 使用模板的可变参数写法，接受不同类型和属相的参数
  template<class T> DeviceKernelArguments(const T *arg)
  {
    add(arg);
  }

  template<class T, class... Args> DeviceKernelArguments(const T *first, Args... args)
  {
    add(first);
    add(args...);
  }
  
  // add函数同样使用模板可变参数，会根据不同的参数类型匹配到不同的add函数
  template<typename T, typename... Args> void add(const T *first, Args... args)
  {
    add(first);
    add(args...);
  }
  void add(const float *value)
  {
    add(FLOAT32, value, sizeof(float));
  }
  void add(const bool *value)
  {
    add(BOOLEAN, value, 4);
  }
  ...
  // 真正用于初始化的add函数
  void add(const Type type, const void *value, size_t size)
  {
    assert(count < MAX_ARGS);

    types[count] = type;
    values[count] = (void *)value;
    sizes[count] = size;
    count++;
  }
}
```

### 主要函数

```
  /* Initialize execution of kernels on this queue.
   *
   * Will, for example, load all data required by the kernels from Device to global or path state.
   *
   * Use this method after device synchronization has finished before enqueueing any kernels. */
  virtual void init_execution() = 0;

  /* Enqueue kernel execution.
   *
   * Execute the kernel work_size times on the device.
   * Supported arguments types:
   * - int: pass pointer to the int
   * - device memory: pass pointer to device_memory.device_pointer
   * Return false if there was an error executing this or a previous kernel. */
  virtual bool enqueue(DeviceKernel kernel,
                       const int work_size,
                       DeviceKernelArguments const &args) = 0;

  /* Wait unit all enqueued kernels have finished execution.
   * Return false if there was an error executing any of the enqueued kernels. */
  virtual bool synchronize() = 0;

  /* Copy memory to/from device as part of the command queue, to ensure
   * operations are done in order without having to synchronize. */
  virtual void zero_to_device(device_memory &mem) = 0;
  virtual void copy_to_device(device_memory &mem) = 0;
  virtual void copy_from_device(device_memory &mem) = 0;
```

这些是DeviceQueue最常使用的函数，通过注释可以理解每个函数的功能，init_execution用于初始化运行环境，enqueue将需要执行的渲染算法加入队列，synchronize用于Host和Device的同步，zero_to_device、copy_to_device、copy_from_device三个是内存管理函数。它们都是纯虚函数，具体实现由每种设备负责。

### CUDADeviceQueue的实现

以CUDADeviceQueue为例，看一下DeviceQueue中主要的几个函数的具体实现。

```
void CUDADeviceQueue::init_execution()
{
  /* Synchronize all textures and memory copies before executing task. */
  CUDAContextScope scope(cuda_device_);
  cuda_device_->load_texture_info();
  cuda_device_assert(cuda_device_, cuCtxSynchronize());

  debug_init_execution();
}
```

init_execution函数初始化CUDAContextScope，并且同步纹理和内存数据。

```
bool CUDADeviceQueue::enqueue(DeviceKernel kernel,
                              const int work_size,
                              DeviceKernelArguments const &args)
{
  if (cuda_device_->have_error()) {
    return false;
  }

  debug_enqueue_begin(kernel, work_size);

  const CUDAContextScope scope(cuda_device_);
  const CUDADeviceKernel &cuda_kernel = cuda_device_->kernels.get(kernel);

  /* Compute kernel launch parameters. */
  const int num_threads_per_block = cuda_kernel.num_threads_per_block;
  const int num_blocks = divide_up(work_size, num_threads_per_block);

  int shared_mem_bytes = 0;

  switch (kernel) {
    case DEVICE_KERNEL_INTEGRATOR_QUEUED_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_QUEUED_SHADOW_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_ACTIVE_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_TERMINATED_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_SORTED_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_COMPACT_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_TERMINATED_SHADOW_PATHS_ARRAY:
    case DEVICE_KERNEL_INTEGRATOR_COMPACT_SHADOW_PATHS_ARRAY:
      /* See parall_active_index.h for why this amount of shared memory is needed. */
      shared_mem_bytes = (num_threads_per_block + 1) * sizeof(int);
      break;

    default:
      break;
  }

  /* Launch kernel. */
  assert_success(cuLaunchKernel(cuda_kernel.function,
                                num_blocks,
                                1,
                                1,
                                num_threads_per_block,
                                1,
                                1,
                                shared_mem_bytes,
                                cuda_stream_,
                                const_cast<void **>(args.values),
                                0),
                 "enqueue");

  debug_enqueue_end();

  return !(cuda_device_->have_error());
}
```

使用Cude执行核心算法，kernel是需要执行的算法。执行CUDA算法时还要分配block的数量和每个block的线程数，如果执行的算法是基于数组的，那么还需要使用shared_mem_bytes指定数组大小[??]。DeviceKernelArguments将会传递到cuLaunchKernel函数中以提供不同算法所需的参数。

```
bool CUDADeviceQueue::synchronize()
{
  if (cuda_device_->have_error()) {
    return false;
  }

  const CUDAContextScope scope(cuda_device_);
  assert_success(cuStreamSynchronize(cuda_stream_), "synchronize");

  debug_synchronize();

  return !(cuda_device_->have_error());
}
```

synchronize使用CUDA内置的函数cuStreamSynchronize。

```
void CUDADeviceQueue::copy_to_device(device_memory &mem)
{
  assert(mem.type != MEM_GLOBAL && mem.type != MEM_TEXTURE);

  if (mem.memory_size() == 0) {
    return;
  }

  /* Allocate on demand. */
  if (mem.device_pointer == 0) {
    cuda_device_->mem_alloc(mem);
  }

  assert(mem.device_pointer != 0);
  assert(mem.host_pointer != nullptr);

  /* Copy memory to device. */
  const CUDAContextScope scope(cuda_device_);
  assert_success(
      cuMemcpyHtoDAsync(
          (CUdeviceptr)mem.device_pointer, mem.host_pointer, mem.memory_size(), cuda_stream_),
      "copy_to_device");
}
```

三个内存管理的函数相似，以copy_to_device为例。传入device_memory，根据不同的操作使用不同的CUDA内存管理函数，比如copy_to_device使用cuMemcpyHtoDAsync，copy_from_device就使用cuMemcpyDtoHAsync。