在渲染过程中，所有的算法都称为Kernel，Kernel意为内核，在CUDA编程中，Kernel是运行在GPU上的C或C++函数，Cycles的中Kernel指的就是运行在Device上的C++渲染算法，这里的Device包括了CPU和GPU。

这里以GPU中的CUDA为例，研究各种Kernel是如何加载到PathWork中并使用的。

## CUDA编程的一些基础知识

CUDA编程让我们可以在GPU上执行一些特定的C或C++函数，一般用于图形处理，深度学习等需要大量并发的场景，因为GPU在多线程场景的执行效率远远高于CPU。CUDA就是用于Nvidia提供的GPU编程工具，全称为Compute Unified Device Architecture。

Nvidia官方文档：[https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)

### 简单的程序示例

```
// Kernel definition
__global__ void VecAdd(float* A, float* B, float* C)
{
    int i = threadIdx.x;
    C[i] = A[i] + B[i];
}

int main()
{
    ...
    // Kernel invocation with N threads
    VecAdd<<<1, N>>>(A, B, C);
    ...
}
```

这里定义了一个向量相加的Kernel，Kernel的定义需要加上__global__标识来声明。其中VecAdd<<<1, N>>>(A, B, C)中的N表示为这个Kernel分配的线程数，比如N = 3，那么这个Kernel就是用于三维向量的相加计算。当执行这个函数时，GPU上分配3个线程同时执行，内置变量threadIdx为线程序号，这样序号为0的线程执行C[0] = A[0] + B[0]，序号为1的线程执行C[1] = A[1] + B[1]，以此类推，可以同时进行N维向量的每个分量的计算。

注意上面的threadIdx.x，其实threadIdx有三个分量x, y和z，因为CUDA中最多支持3维线程的分配。事实上，CUDA的线程分配分为线程组和线程数，block表示分配多少个线程组，thread表示每个线程组中的线程数量，它们的分配都可以支持最多3维，比如：

```
__global__ void MatAdd(float A[N][N], float B[N][N],
float C[N][N])
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    int j = blockIdx.y * blockDim.y + threadIdx.y;
    if (i < N && j < N)
        C[i][j] = A[i][j] + B[i][j];
}

int main()
{
    ...
    // Kernel invocation
    dim3 threadsPerBlock(16, 16);
    dim3 numBlocks(N / threadsPerBlock.x, N / threadsPerBlock.y);
    MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);
    ...
}
```

MatAdd<<<numBlocks, threadsPerBlock>>>表示分配numBlocks个线程组，每个线程组有threadsPerBlock个线程，假设N = 32，则numBlocks = (2, 2)，那么上述代码表示分配2 * 2 = 4个block，每个block有16 * 16 = 256个线程，一共分配了4 * 256 = 1024个线程。再看函数MatAdd，N = 32时，它用于计算规模为32 * 32的两个矩阵相加，计算线程索引 i 和 j 时，blockIdx表示线程组的索引，blockDim表示线程组的维度，比如在这里blockDim.x = blockDim.y = 16，threadIdx还是线程的索引。这样就可以同时计算 A[0][0] + B[0][0] 到 A[31][31] + B[31][31] 的结果。

### Device Memory

顾名思义，Device Memory是Device上的内存，也就是GPU上的内存。在使用GPU计算的过程中，不可避免地需要对程序的内存进行管理，CUDA提供了类似于C和C++语言的内存管理接口，包括内存的分配和释放，内存的拷贝，在Host和Device之间进行数据传输等。

一些常用的Device Memory管理接口：

- cudaMalloc：分配内存，对应malloc函数
    
- cudaFree：释放内存，对应free函数
    
- cudaMemcpy：内存拷贝，对应memcpy，可以在Host和Device之间拷贝数据
    
- cudaMallocPitch：分配内存，用于分配二维的数据
    
- cudaMalloc3D：分配内存，用于分配三维的数据
    
- cudaMemcpyToSymbol：将内存数据拷贝到一个全局变量
    
- cudaMemcpyFromSymbol：将一个全局变量拷贝到内存
    
- cudaGetSymbolAddress：获得一个全局变量的地址
    

在CUDA编程中，由于使用的内存位于GPU，全局变量的使用没有普通程序自由，需要使用cudaMemcpyToSymbol这些函数才能设置和获得全局变量的值。

#### Global Memory

全局内存可以被所有线程访问，但是它的访问速度是最慢的，通常用于储存输入和输出数据。平时使用的cudaMalloc就是在全局内存上分配空间。全局内存变量的指定不需要加任何内存空间指定的声明，它在整个程序运行期间存在。可以使用cudaMemcpyToSymbol这些函数访问位于全局内存的变量。

#### Shared Memory

共享内存的访问速度比全局内存更快，如果计算时数据会被频繁地读取，应该把这些数据存入共享内存中。共享内存中的变量用__shared__声明，它们只能被同一个block的线程访问，生存周期也仅限于block。

#### Constant Memory

常量内存用于保存一些只读的数据，这些数据存于一个缓存中，在读取时拥有更好的性能。常量数据用__constant__声明，他和全局内存一样，可以被所有线程访问，生命周期为整个程序。

#### Texture and Surface Memory

纹理与表面的内存是一种专用内存，用于储存纹理和表面数据。纹理数据可以是一维，二维或者三维的数组，数组中的每一个元素都是一个texel，每个texel包含4个浮点数，对应颜色的r, g, b, a分量。同时，还支持addressMode和filterMode等纹理读取设置，也提供个纹理读写的API，比如使用tex2D\<float\>(texObj, u, v)来读取一张图片(u, v)处的纹理数据。表面数据与纹理数据相似，可用来储存像素，几何数据，体数据等，与纹理数据不同的是，表面数据的读取需要在x分量上乘以像素大小，比如，surf2Dread(value, surf, x * sizeof(float4), y)读取(x, y)处的表面数据。

### C++ 支持

CUDA提供了各种内置函数、关键字和变量来简化C++编写kernel的难度，被称为C++语言扩展(C++ Language Extensions)，这里介绍一下常用的一些常用的扩展。

#### 函数执行空间指定

用于指定此函数是在Host还是Device上执行，或者是否可以从Host或Device调用。这些关键字在声明函数时函数的返回值前使用。

__global__ 表示这个函数是一个Kernel，它在Device上执行，可以从Host调用它。它的返回值必须为void，并且不能成为类的成员函数。Kernel的执行是异步的，也就是调用操作结束后它可能还没有在Device上执行完。

__device__ 表示此函数在Device上执行，并且只能在Device上调用。

__host__ 表示此函数在Host上执行，并且只能在Host上调用，一般可以省略。

__device__和__host__可以在同一个函数使用，表示此函数在Host和Device上都需要编译，都可以使用。

__noinline__表示此函数不能编译为内联函数，默认情况下__device__指定的函数会被编译成内联函数。

__forceinline__表示强制让此函数编译为内联函数。

#### 变量内存空间指定

\_\_device\_\_ 表示此变量位于Device中的Global Memory内，也是Device中的默认内存空间。

\_\_constant\_\_和\_\_device\_\_一起使用，表示此变量位于Constant Memory中。

\_\_shared\_\_和\_\_device\_\_一起使用，表示此变量位于Shared Memory中。

__managed__和__device__一起使用，表示此变量在Host和Device上都可以使用，并且生存周期为整个程序。

__restrict__ 是一个指针类型的限定符，相当于C类型语言中的restrict指针，它告诉编译器这个指针是访问

其所指向内存区域的唯一指针，这样编译器就可以采取一些优化措施来提升性能，比如将此内存的数据放入寄存器中。

#### TextureAPI

cudaTextureObject_t 是CUDA中的纹理对象，使用cudaCreateTextureObject()函数创建。纹理对象提供了更加友好的纹理图片操作方法。

tex1D(cudaTextureObject_t texObj, float x)表示从一维纹理texObj中获取x处的像素。

tex1DLod(cudaTextureObject_t texObj, float x, float level)表示从一维纹理texObj中获取level的LOD层级中x处的像素。

tex1DGrad(cudaTextureObject_t texObj, float x, float dx, float dy)和tex1DLod相似，只是它的LOD有dx和dy指定。

同理，2D和3D的纹理也有这些操作函数，只是参数从一维的x变成了二维的(x, y)和三维的(x, y, z)。

### cuLaunchKernel

除了使用实例中的Kernel_function<<<numBlocks, threadsPerBlock>>>(args...)方式来调用kernel外，CUDA中还可以使用cuLaunchKernel函数来调用kernel，cuLaunchKernel原型为

```
CUresult cuLaunchKernel(CUfunction f, unsigned int gridDimX, unsigned int gridDimY, unsigned int gridDimZ, unsigned int blockDimX, unsigned int blockDimY, unsigned int blockDimZ, unsigned int sharedMemBytes, CUstream hStream, void **kernelParams, void **extra)
```

- f：要启动的内核函数的句柄。
    
- gridDimX、gridDimY、gridDimZ：内核函数的网格维度，即网格中的线程块数量。
    
- blockDimX、blockDimY、blockDimZ：内核函数的线程块维度，即每个线程块中的线程数量。
    
- sharedMemBytes：指定每个线程块中共享内存的大小（以字节为单位）。
    
- hStream：指定用于执行内核函数的流。 cuLaunchKernel函数会根据给定的参数在GPU上启动一个内核函数，并将其放在流。 即该Kernel使用的参数。
    

## Cycles中的宏定义

src\kernel\device\cuda\compat.h文件

```
#define ccl_device __device__ __inline__
#define ccl_device_extern extern "C" __device__
#  define ccl_device_inline __device__ __inline__
#  define ccl_device_forceinline __device__ __forceinline__
#define ccl_device_noinline __device__ __noinline__
#define ccl_device_noinline_cpu ccl_device
#define ccl_device_inline_method ccl_device
#define ccl_global
#define ccl_inline_constant __constant__
#define ccl_device_constant __constant__ __device__
#define ccl_constant const
#define ccl_gpu_shared __shared__
#define ccl_private
#define ccl_may_alias
#define ccl_restrict __restrict__
```

这几个宏定义了CUDA中常用的函数和变量限定符，从名称上更加容易理解，加上了统一的ccl前缀。

```
#define ccl_gpu_thread_idx_x (threadIdx.x)
#define ccl_gpu_block_dim_x (blockDim.x)
#define ccl_gpu_block_idx_x (blockIdx.x)
#define ccl_gpu_grid_dim_x (gridDim.x)
#define ccl_gpu_warp_size (warpSize)
#define ccl_gpu_thread_mask(thread_warp) uint(0xFFFFFFFF >> (ccl_gpu_warp_size - thread_warp))

#define ccl_gpu_global_id_x() (ccl_gpu_block_idx_x * ccl_gpu_block_dim_x + ccl_gpu_thread_idx_x)
#define ccl_gpu_global_size_x() (ccl_gpu_grid_dim_x * ccl_gpu_block_dim_x)
```

这些宏定义CUDA中内置的block和线程变量，blockIdx，blockDim，threadIdx分别是CUDA中的block索引，block大小，block中的thread索引，这里只定义了获取x维度索引的宏。

```
typedef unsigned long long CUtexObject;
typedef CUtexObject ccl_gpu_tex_object_2D;
typedef CUtexObject ccl_gpu_tex_object_3D;

template<typename T>
ccl_device_forceinline T ccl_gpu_tex_object_read_2D(const ccl_gpu_tex_object_2D texobj,
                                                    const float x,
                                                    const float y)
{
  return tex2D<T>(texobj, x, y);
}
```

这里将unsigned long long定义为CUtexObject，没有使用CUDA中的cudaTextureObject_t 对象，并且定义了它的读取函数（这里以2D为例）。可能与使用的第三方库cuew有关。

src\kernel\device\cuda\config.h文件

```
#define ccl_gpu_kernel(block_num_threads, thread_num_registers) \
  extern "C" __global__ void __launch_bounds__(block_num_threads, \
                                               GPU_MULTIPRESSOR_MAX_REGISTERS / \
                                                   (block_num_threads * thread_num_registers))
#define ccl_gpu_kernel_threads(block_num_threads) \
  extern "C" __global__ void __launch_bounds__(block_num_threads)
```

这里使用的__launch_bounds__用法为：

```
__global__ void
__launch_bounds__(maxThreadsPerBlock, minBlocksPerMultiprocessor, maxBlocksPerCluster)
MyKernel(...)
{
    ...
}
```

指定kernel的block和thread数量，这是一种声明Kernel时的性能优化方法，编译器会根据3个参数最小化register和指令的生成。（比较复杂，现在只当它是在声明Kernel）

```
#define ccl_gpu_kernel_signature(name, ...) kernel_gpu_##name(__VA_ARGS__)
#define ccl_gpu_kernel_postfix

#define ccl_gpu_kernel_call(x) x
```

简化kernel声明时的名称使用，自动加上kernel_gpu前缀；强调kernel的调用。

```
#define ccl_gpu_kernel_lambda(func, ...) \
  struct KernelLambda { \
    __VA_ARGS__; \
    __device__ int operator()(const int state) \
    { \
      return (func); \
    } \
  } ccl_gpu_kernel_lambda_pass
```

定义一个函数对象，其中"func"是lambda主体，而额外的参数用于指定捕获的状态state。

src\kernel\device\cuda\global.h文件

```
#define kernel_data kernel_params.data
#define kernel_data_fetch(name, index) kernel_params.name[(index)]
#define kernel_data_array(name) (kernel_params.name)
#define kernel_integrator_state kernel_params.integrator_state
```

简化kernel_params参数的使用，kernel_params是一个常量，保存了场景信息，纹理信息，积分器的信息等。

## Kernel的定义和加载

以integrator_init_from_camera为例，研究Cycles中的kernel是如何定义的。

首先，src\kernel\types.h中定义了如下枚举类型：

```
typedef enum DeviceKernel : int {
  DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA = 0,
  DEVICE_KERNEL_INTEGRATOR_INIT_FROM_BAKE,
  DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST,
  ...
  DEVICE_KERNEL_NUM
}
```

DeviceKernel 中的每一个枚举对应了一个kernel。

同时，在src\device\kernel.cpp中定义了如何获得每个枚举的字符串标识：

```
const char *device_kernel_as_string(DeviceKernel kernel)
{
  switch (kernel) {
    /* Integrator. */
    case DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA:
      return "integrator_init_from_camera";
    case DEVICE_KERNEL_INTEGRATOR_INIT_FROM_BAKE:
      return "integrator_init_from_bake";
    case DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST:
      return "integrator_intersect_closest";
    ...
}
```

DEVICE_KERNEL_INTEGRATOR_INIT_FROM_CAMERA对应的字符串为integrator_init_from_camera。

在src\device\cuda\device_impl.cpp中定义了load_kernels函数，这个函数用于加载kernel，

```
bool CUDADevice::load_kernels(const uint kernel_features)
{
  /* TODO(sergey): Support kernels re-load for CUDA devices adaptive compile.
   *
   * Currently re-loading kernel will invalidate memory pointers,
   * causing problems in cuCtxSynchronize.
   */
  if (cuModule) {
    if (use_adaptive_compilation()) {
      VLOG_INFO
          << "Skipping CUDA kernel reload for adaptive compilation, not currently supported.";
    }
    return true;
  }

  /* check if cuda init succeeded */
  if (cuContext == 0)
    return false;

  /* check if GPU is supported */
  if (!support_device(kernel_features))
    return false;

  /* get kernel */
  const char *kernel_name = "kernel";
  string cflags = compile_kernel_get_common_cflags(kernel_features);
  string cubin = compile_kernel(cflags, kernel_name);
  if (cubin.empty())
    return false;

  /* open module */
  CUDAContextScope scope(this);

  string cubin_data;
  CUresult result;

  if (path_read_text(cubin, cubin_data))
    result = cuModuleLoadData(&cuModule, cubin_data.c_str());
  else
    result = CUDA_ERROR_FILE_NOT_FOUND;

  if (result != CUDA_SUCCESS)
    set_error(string_printf(
        "Failed to load CUDA kernel from '%s' (%s)", cubin.c_str(), cuewErrorString(result)));

  if (result == CUDA_SUCCESS) {
    kernels.load(this);
    reserve_local_memory(kernel_features);
  }

  return (result == CUDA_SUCCESS);
}
```

此函数传入的参数为kernel_features，它是一个uint类型，其中的每一位对应了Cycle中的一个Feature。这里通过kernel_features计算CUmodule的路径，然后通过路径使用cuModuleLoadData函数将其读入cuModule变量中，这就像在Windows上使用C++读取dll库一样。读取CUmodule完成后，kernels.load(this)将所有的kernel加载到运行的程序中。kernels.load(this)的原型为

```
void CUDADeviceKernels::load(CUDADevice *device)
{
  CUmodule cuModule = device->cuModule;

  for (int i = 0; i < (int)DEVICE_KERNEL_NUM; i++) {
    CUDADeviceKernel &kernel = kernels_[i];
    
    /* No mega-kernel used for GPU. */
    if (i == DEVICE_KERNEL_INTEGRATOR_MEGAKERNEL) {
      continue;
    }

    const std::string function_name = std::string("kernel_gpu_") +
                                      device_kernel_as_string((DeviceKernel)i);
    cuda_device_assert(device,
                       cuModuleGetFunction(&kernel.function, cuModule, function_name.c_str()));

    if (kernel.function) {
      cuda_device_assert(device, cuFuncSetCacheConfig(kernel.function, CU_FUNC_CACHE_PREFER_L1));

      cuda_device_assert(
          device,
          cuOccupancyMaxPotentialBlockSize(
              &kernel.min_blocks, &kernel.num_threads_per_block, kernel.function, NULL, 0, 0));
    }
    else {
      LOG(ERROR) << "Unable to load kernel " << function_name;
    }
  }

  loaded = true;
}
```
这里遍历所有DeviceKernel,使用cuModuleGetFunction函数将其加载到kernel.function中，每一个kernel的名称为("kernel_gpu_") + device_kernel_as_string(kernel)，比如 integrator_init_from_camera的全名称因该是kernel_gpu_integrator_init_from_camera，这正好对应了上面的ccl_gpu_kernel_signature宏定义，它会自动在kernel名称前加上kernel_gpu_。
load_kernels在session初始化的scene初始化阶段进行。
以integrator_init_from_camera为例，了解一个kernel加载完成后是如何被调用的：
```
ccl_gpu_kernel(GPU_KERNEL_BLOCK_NUM_THREADS, GPU_KERNEL_MAX_REGISTERS)
    ccl_gpu_kernel_signature(integrator_init_from_camera,
                             ccl_global KernelWorkTile *tiles,
                             const int num_tiles,
                             ccl_global float *render_buffer,
                             const int max_tile_work_size)
{
  const int work_index = ccl_gpu_global_id_x();

  if (work_index >= max_tile_work_size * num_tiles) {
    return;
  }

  const int tile_index = work_index / max_tile_work_size;
  const int tile_work_index = work_index - tile_index * max_tile_work_size;

  ccl_global const KernelWorkTile *tile = &tiles[tile_index];

  if (tile_work_index >= tile->work_size) {
    return;
  }

  const int state = tile->path_index_offset + tile_work_index;

  uint x, y, sample;
  ccl_gpu_kernel_call(get_work_pixel(tile, tile_work_index, &x, &y, &sample));

  ccl_gpu_kernel_call(
      integrator_init_from_camera(nullptr, state, tile, render_buffer, x, y, sample));
}
```
结合上面的宏定义，ccl_gpu_kernel用于指定Device上为该kernel分配的block和thread数量，ccl_gpu_kernel_signature为integrator_init_from_camera加上kernel_gpu_前缀，其余的参数才是此定义函数的实参。

work_index是线程的索引，使用work_index计算当前线程所在tile的tile_index，tile_work_index则是此线程在当前tile的偏移量。比如work_index = 66，max_tile_work_size = 64，tile_index = 66 / 64 = 1，tile_work_index = 66 - 1 * 64 = 2。

通过tile_index取出当前的tile，path_index_offset是在PathTraceWork预计算好的，表示当前tile的path偏移，比如一共有2个tile，每个tile由64个pixel，每个pixel有2个path，那么第一个tile的path_index_offset = num_active_paths ，第二个tile的path_index_offset = tile1.path_index_offset + 64 * 2，state表示当前path或者说线程的偏移量。

然后调用get_work_pixel获得该tile的最终采样的x, y坐标和sample（这里的线程分配是3维吗），最后调用Device上的integrator_init_from_camera函数进行计算。