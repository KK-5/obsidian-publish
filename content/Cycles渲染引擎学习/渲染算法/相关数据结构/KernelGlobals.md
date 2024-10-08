## KernelGlobals[CPU渲染时使用]

```
struct KernelGlobalsGPU {
  int unused[1];
};
typedef ccl_global const KernelGlobalsGPU *ccl_restrict KernelGlobals
```

```
typedef struct KernelGlobalsCPU {
#define KERNEL_DATA_ARRAY(type, name) kernel_array<type> name;
#include "kernel/data_arrays.h"
  KernelData data;
#ifdef __OSL__
  /* On the CPU, we also have the OSL globals here. Most data structures are shared
   * with SVM, the difference is in the shaders and object/mesh attributes. */
  OSLGlobals *osl = nullptr;
  OSLShadingSystem *osl_ss = nullptr;
  OSLThreadData *osl_tdata = nullptr;
#endif
#ifdef __PATH_GUIDING__
  /* Pointers to global data structures. */
  openpgl::cpp::SampleStorage *opgl_sample_data_storage = nullptr;
  openpgl::cpp::Field *opgl_guiding_field = nullptr;
  /* Local data structures owned by the thread. */
  openpgl::cpp::PathSegmentStorage *opgl_path_segment_storage = nullptr;
  openpgl::cpp::SurfaceSamplingDistribution *opgl_surface_sampling_distribution = nullptr;
  openpgl::cpp::VolumeSamplingDistribution *opgl_volume_sampling_distribution = nullptr;
#endif
  /* **** Run-time data ****  */
  ProfilingState profiler;
} KernelGlobalsCPU;
typedef const KernelGlobalsCPU *ccl_restrict KernelGlobals;
```

KernelGlobals只用于CPU，GPU仅有定义，没有真正使用，KernelGlobals的核心是KernelData，还包含性能统计、OSL和PATH_GUIDING的信息。

KernelData定义如下：

```
typedef struct KernelData {
  /* Features and limits. */
  uint kernel_features;
  uint max_closures;
  uint max_shaders;
  uint volume_stack_size;
  /* Always dynamic data members. */
  KernelCamera cam;
  KernelBake bake;
  KernelTables tables;
  /* Potentially specialized data members. */
#define KERNEL_STRUCT_BEGIN(name, parent) name parent;
#include "kernel/data_template.h"
  /* Device specific BVH. */
#ifdef __KERNEL_OPTIX__
  OptixTraversableHandle device_bvh;
#elif defined __METALRT__
  metalrt_as_type device_bvh;
#else
#  ifdef __EMBREE__
  RTCScene device_bvh;
#    ifndef __KERNEL_64_BIT__
  int pad1;
#    endif
#  else
  int device_bvh, pad1;
#  endif
#endif
  int pad2, pad3;
} KernelData;
```

KernelData包含了渲染的配置、渲染相机（或Bake信息）以及BVH数据。

KernelData中包含了一个kernel/data_template.h，在其中定义了KernelBackground、KernelBVH、KernelFilm、KernelIntegrator和KernelSVMUsage的结构体类型，也是渲染时需要的数据。