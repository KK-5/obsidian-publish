在GPU编程中，很多状态信息需要在不同的内核执行之间进行传递和保留，以便实现高效的并行计算和数据共享。IntegratorState意为积分器的状态，他作用就是可以在不同Kernel之间进行信息共享，IntegratorState不能占用太多内存，否则会影响执行效率，他的内存管理和分配由不同的硬件来决定。

# IntegratorShadowStateCPU

```
typedef struct IntegratorShadowStateCPU {
#define KERNEL_STRUCT_BEGIN(name) struct {
#define KERNEL_STRUCT_MEMBER(parent_struct, type, name, feature) type name;
#define KERNEL_STRUCT_ARRAY_MEMBER KERNEL_STRUCT_MEMBER
#define KERNEL_STRUCT_END(name) \
  } \
  name;
#define KERNEL_STRUCT_END_ARRAY(name, cpu_size, gpu_size) \
  } \
  name[cpu_size];
#define KERNEL_STRUCT_VOLUME_STACK_SIZE MAX_VOLUME_STACK_SIZE
#include "kernel/integrator/shadow_state_template.h"
#undef KERNEL_STRUCT_BEGIN
#undef KERNEL_STRUCT_MEMBER
#undef KERNEL_STRUCT_ARRAY_MEMBER
#undef KERNEL_STRUCT_END
#undef KERNEL_STRUCT_END_ARRAY
} IntegratorShadowStateCPU;
typedef struct IntegratorStateCPU {
#define KERNEL_STRUCT_BEGIN(name) struct {
#define KERNEL_STRUCT_MEMBER(parent_struct, type, name, feature) type name;
#define KERNEL_STRUCT_ARRAY_MEMBER KERNEL_STRUCT_MEMBER
#define KERNEL_STRUCT_END(name) \
  } \
  name;
#define KERNEL_STRUCT_END_ARRAY(name, cpu_size, gpu_size) \
  } \
  name[cpu_size];
#define KERNEL_STRUCT_VOLUME_STACK_SIZE MAX_VOLUME_STACK_SIZE
#include "kernel/integrator/state_template.h"
#undef KERNEL_STRUCT_BEGIN
#undef KERNEL_STRUCT_MEMBER
#undef KERNEL_STRUCT_ARRAY_MEMBER
#undef KERNEL_STRUCT_END
#undef KERNEL_STRUCT_END_ARRAY
#undef KERNEL_STRUCT_VOLUME_STACK_SIZE
  IntegratorShadowStateCPU shadow;
  IntegratorShadowStateCPU ao;
} IntegratorStateCPU;
```

用于CPU渲染时的IntegratorState，这里不详细研究。

# IntegratorStateGPU

```
typedef struct IntegratorStateGPU {
#define KERNEL_STRUCT_BEGIN(name) struct {
#define KERNEL_STRUCT_MEMBER(parent_struct, type, name, feature) ccl_global type *name;
#define KERNEL_STRUCT_ARRAY_MEMBER KERNEL_STRUCT_MEMBER
#define KERNEL_STRUCT_END(name) \
  } \
  name;
#define KERNEL_STRUCT_END_ARRAY(name, cpu_size, gpu_size) \
  } \
  name[gpu_size];
#define KERNEL_STRUCT_VOLUME_STACK_SIZE MAX_VOLUME_STACK_SIZE
#include "kernel/integrator/state_template.h"
#include "kernel/integrator/shadow_state_template.h"
#undef KERNEL_STRUCT_BEGIN
#undef KERNEL_STRUCT_MEMBER
#undef KERNEL_STRUCT_ARRAY_MEMBER
#undef KERNEL_STRUCT_END
#undef KERNEL_STRUCT_END_ARRAY
#undef KERNEL_STRUCT_VOLUME_STACK_SIZE
  /* Count number of queued kernels. */
  ccl_global IntegratorQueueCounter *queue_counter;
  /* Count number of kernels queued for specific shaders. */
  ccl_global int *sort_key_counter[DEVICE_KERNEL_INTEGRATOR_NUM];
  /* Index of shadow path which will be used by a next shadow path.  */
  ccl_global int *next_shadow_path_index;
  /* Index of main path which will be used by a next shadow catcher split.  */
  ccl_global int *next_main_path_index;
  /* Partition/key offsets used when writing sorted active indices. */
  ccl_global int *sort_partition_key_offsets;
  /* Divisor used to partition active indices by locality when sorting by material.  */
  uint sort_partition_divisor;
} IntegratorStateGPU;
```

## KERNEL_STRUCT宏定义

在IntegratorState中出现了KERNEL_STRUCT_BEGIN这样的宏定义，而后又会将其undef ，它是一个辅助的宏，用来定义IntegratorState中的各种属性。

IntegratorState中这种定义数据的格式在项目中经常使用，首先定义宏：

```
#define KERNEL_STRUCT_BEGIN(name) struct {
#define KERNEL_STRUCT_MEMBER(parent_struct, type, name, feature) ccl_global type *name;
#define KERNEL_STRUCT_END(name) \
  } \
  name;
```

然后就可以用这个宏做结构体的定义：

```
KERNEL_STRUCT_BEGIN(path)
KERNEL_STRUCT_MEMBER(path, uint32_t, sample, KERNEL_FEATURE_PATH_TRACING)
KERNEL_STRUCT_END(path)
```

上面的代码就相当于：

```
struct {
    ccl_global uint32_t *sample;
} path;
```

这样在需要定义大量的结构体时，可以单独把他们放到另一个文件中，再引用过来。

在parent_struct, type, name, feature这几份宏定义的参数中，看起来feature是没有用到的，只起到一个标识作用。

以IntegratorStateGPU为例，它include了两个文件state_template.h和shadow_state_template.h。state_template.h中定义了的Path、Ray、Intersection、Subsurface等结构体类型，shadow_state_template.h定义了shadow_path、shadow_ray等结构体类型。

state_template.h文件的一些定义：

```
KERNEL_STRUCT_BEGIN(path)
/* Index of a pixel within the device render buffer where this path will write its result.
 * To get an actual offset within the buffer the value needs to be multiplied by the
 * `kernel_data.film.pass_stride`.
 *
 * The multiplication is delayed for later, so that state can use 32bit integer. */
KERNEL_STRUCT_MEMBER(path, uint32_t, render_pixel_index, KERNEL_FEATURE_PATH_TRACING)
/* Current sample number. */
KERNEL_STRUCT_MEMBER(path, uint32_t, sample, KERNEL_FEATURE_PATH_TRACING)
/* Current ray bounce depth. */
KERNEL_STRUCT_MEMBER(path, uint16_t, bounce, KERNEL_FEATURE_PATH_TRACING)
/* Current transparent ray bounce depth. */
KERNEL_STRUCT_MEMBER(path, uint16_t, transparent_bounce, KERNEL_FEATURE_PATH_TRACING)
/* Current diffuse ray bounce depth. */
KERNEL_STRUCT_MEMBER(path, uint16_t, diffuse_bounce, KERNEL_FEATURE_PATH_TRACING)
/* Current glossy ray bounce depth. */
KERNEL_STRUCT_MEMBER(path, uint16_t, glossy_bounce, KERNEL_FEATURE_PATH_TRACING)
...
KERNEL_STRUCT_END(path)
```

这里就是定义了一个path结构体，它的成员有render_pixel_index，sample，bounce等。

## INTEGRATOR_STATE宏定义

```
#  define INTEGRATOR_STATE(state, nested_struct, member) \
    kernel_integrator_state.nested_struct.member[state]
#  define INTEGRATOR_STATE_WRITE(state, nested_struct, member) \
    INTEGRATOR_STATE(state, nested_struct, member)

#  define INTEGRATOR_STATE_ARRAY(state, nested_struct, array_index, member) \
    kernel_integrator_state.nested_struct[array_index].member[state]
#  define INTEGRATOR_STATE_ARRAY_WRITE(state, nested_struct, array_index, member) \
    INTEGRATOR_STATE_ARRAY(state, nested_struct, array_index, member)
```

这些宏定义是用来快速地读取或写入state中的属性的，比如想读取一个state的path中的sample属性：

```
sample = INTEGRATOR_STATE(state, path, sample)
// 等价于 sample = kernel_integrator_state.path.sample[state]
```

同理，写入此属性:

```
INTEGRATOR_STATE_WRITE(state, path, sample) = sample
// 等价于 kernel_integrator_state.path.sample[state] = sample
```

如果是数组成员的话加上一个array_index参数。

需要注意的是，IntegratorStateGPU和IntegratorStateCPU关于INTEGRATOR_STATE宏的定义是不一样的，IntegratorStateCPU中是这样定义的：

```
#  define INTEGRATOR_STATE(state, nested_struct, member) ((state)->nested_struct.member)
#  define INTEGRATOR_STATE_WRITE(state, nested_struct, member) ((state)->nested_struct.member)

#  define INTEGRATOR_STATE_ARRAY(state, nested_struct, array_index, member) \
    ((state)->nested_struct[array_index].member)
#  define INTEGRATOR_STATE_ARRAY_WRITE(state, nested_struct, array_index, member) \
    ((state)->nested_struct[array_index].member)
```

它的state是一个结构体，以上面读取path中的sample属性为例：

```
sample = INTEGRATOR_STATE(state, path, sample)
// 等价于 sample = ((state)->path.sample)
```

在GPU中，state是一个索引，从kernel_integrator_state中根据此索引取数据，在CPU中，state是一个完整的结构体，可以直接从它的内部取数据。

在CPU中，IntegratorState的定义是这样的：

```
typedef IntegratorStateCPU *ccl_restrict IntegratorState;
typedef const IntegratorStateCPU *ccl_restrict ConstIntegratorState;
typedef IntegratorShadowStateCPU *ccl_restrict IntegratorShadowState;
typedef const IntegratorShadowStateCPU *ccl_restrict ConstIntegratorShadowState;
```

直接将IntegratorStateCPU作为IntegratorState，

在GPU中，IntegratorState的定义是这样的：

```
typedef int IntegratorState;
typedef int ConstIntegratorState;
typedef int IntegratorShadowState;
typedef int ConstIntegratorShadowState;
```

IntegratorState就是一个int值。在CUDA中，这个IntegratorState是线程的索引，每个线程计算光线追踪的一条path。

## kernel_integrator_state

通过上面对于IntegratorState的分析可以知道，GPU上的IntegratorState数据是通过索引从kernel_integrator_state获取的。这里以CUDA为例，了解kernel_integrator_state是如何定义的，包含了哪些内容。

在 src\kernel\device\cuda\globals.h 文件中，可以看到如下定义：

```
struct KernelParamsCUDA {
  /* Global scene data and textures */
  KernelData data;
#define KERNEL_DATA_ARRAY(type, name) const type *name;
#include "kernel/data_arrays.h"

  /* Integrator state */
  IntegratorStateGPU integrator_state;
};

#ifdef __KERNEL_GPU__
__constant__ KernelParamsCUDA kernel_params;
#endif

/* Abstraction macros */
#define kernel_data kernel_params.data
#define kernel_data_fetch(name, index) kernel_params.name[(index)]
#define kernel_data_array(name) (kernel_params.name)
#define kernel_integrator_state kernel_params.integrator_state
```

从这里可以看到kernel_integrator_state是一个IntegratorStateGPU类型的数据，它保存在KernelParamsCUDA结构体中。kernel_params是KernelParamsCUDA结构体的实例，它在初始化完成后将数据copy到GPU。

## 辅助函数

在src\kernel\integrator\state_util.h文件中定义了一些辅助函数，它利用INTEGRATOR_STATE的宏定义，将IntegratorState中对一个结构体的读写操作封装到一起，比如下面就是对IntegratorState的Ray数据的读写操作：

```
/* Ray */

ccl_device_forceinline void integrator_state_write_ray(IntegratorState state,
                                                       ccl_private const Ray *ccl_restrict ray)
{
  INTEGRATOR_STATE_WRITE(state, ray, P) = ray->P;
  INTEGRATOR_STATE_WRITE(state, ray, D) = ray->D;
  INTEGRATOR_STATE_WRITE(state, ray, tmin) = ray->tmin;
  INTEGRATOR_STATE_WRITE(state, ray, tmax) = ray->tmax;
  INTEGRATOR_STATE_WRITE(state, ray, time) = ray->time;
  INTEGRATOR_STATE_WRITE(state, ray, dP) = ray->dP;
  INTEGRATOR_STATE_WRITE(state, ray, dD) = ray->dD;
}

ccl_device_forceinline void integrator_state_read_ray(ConstIntegratorState state,
                                                      ccl_private Ray *ccl_restrict ray)
{
  ray->P = INTEGRATOR_STATE(state, ray, P);
  ray->D = INTEGRATOR_STATE(state, ray, D);
  ray->tmin = INTEGRATOR_STATE(state, ray, tmin);
  ray->tmax = INTEGRATOR_STATE(state, ray, tmax);
  ray->time = INTEGRATOR_STATE(state, ray, time);
  ray->dP = INTEGRATOR_STATE(state, ray, dP);
  ray->dD = INTEGRATOR_STATE(state, ray, dD);
}
```