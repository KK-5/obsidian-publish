按照path的计算顺序，camera ray生成后的下一步就是在场景中计算第一个碰撞到的物体，也就是执行integrator_intersect_closest这个kernel，然而kernel的执行并不一定是按照path的计算顺序指行的，根据CPU侧的调度算法，每一次执行的都是path数量最多的kernel，换句话说就是有最多的path需要执行的kernel，这样对GPU架构更加友好，可以提高渲染性能。
不过这里依然按照path的计算顺序来分析每个kernel的算法。
integrator_intersect_closest源码在 src\kernel\integrator\intersect_closest.h

```
ccl_device void integrator_intersect_closest(KernelGlobals kg,
                                             IntegratorState state,
                                             ccl_global float *ccl_restrict render_buffer)
{
  PROFILING_INIT(kg, PROFILING_INTERSECT_CLOSEST);

  /* Read ray from integrator state into local memory. */
  Ray ray ccl_optional_struct_init;
  integrator_state_read_ray(state, &ray);
  kernel_assert(ray.tmax != 0.0f);

  const uint visibility = path_state_ray_visibility(state);
  const int last_isect_prim = INTEGRATOR_STATE(state, isect, prim);
  const int last_isect_object = INTEGRATOR_STATE(state, isect, object);

  /* Trick to use short AO rays to approximate indirect light at the end of the path. */
  if (path_state_ao_bounce(kg, state)) {
    ray.tmax = kernel_data.integrator.ao_bounces_distance;

    if (last_isect_object != OBJECT_NONE) {
      const float object_ao_distance = kernel_data_fetch(objects, last_isect_object).ao_distance;
      if (object_ao_distance != 0.0f) {
        ray.tmax = object_ao_distance;
      }
    }
  }

  /* Scene Intersection. */
  Intersection isect ccl_optional_struct_init;
  isect.object = OBJECT_NONE;
  isect.prim = PRIM_NONE;
  ray.self.object = last_isect_object;
  ray.self.prim = last_isect_prim;
  ray.self.light_object = OBJECT_NONE;
  ray.self.light_prim = PRIM_NONE;
  ray.self.light = LAMP_NONE;
  bool hit = scene_intersect(kg, &ray, visibility, &isect);

  /* TODO: remove this and do it in the various intersection functions instead. */
  if (!hit) {
    isect.prim = PRIM_NONE;
  }

  /* Setup mnee flag to signal last intersection with a caster */
  const uint32_t path_flag = INTEGRATOR_STATE(state, path, flag);

#ifdef __MNEE__
  /* Path culling logic for MNEE (removes fireflies at the cost of bias) */
  if (kernel_data.integrator.use_caustics) {
    ...
  }
#endif /* MNEE*/

  /* Light intersection for MIS. */
  if (kernel_data.integrator.use_light_mis && !integrator_intersect_skip_lights(kg, state)) {
    /* NOTE: if we make lights visible to camera rays, we'll need to initialize
     * these in the path_state_init. */
    const int last_type = INTEGRATOR_STATE(state, isect, type);
    hit = lights_intersect(
              kg, state, &ray, &isect, last_isect_prim, last_isect_object, last_type, path_flag) ||
          hit;
  }

  /* Write intersection result into global integrator state memory. */
  integrator_state_write_isect(state, &isect);

  /* Setup up next kernel to be executed. */
  integrator_intersect_next_kernel<DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST>(
      kg, state, &isect, render_buffer, hit);
}
```
MNEE全称为 Manifold Next Event Estimation，流形下一事件估计，用来计算焦散的，这里我们先忽略掉它。
依然按照每个阶段分析kernel算法，简单的流程就跳过。
## 检测光线的可见性

```
  const uint visibility = path_state_ray_visibility(state);
  // 如果path是刚从camera ray, last_isect_prim和last_isect_object为空
  const int last_isect_prim = INTEGRATOR_STATE(state, isect, prim);
  const int last_isect_object = INTEGRATOR_STATE(state, isect, object);
```

主要查看path_state_ray_visibility的实现，源码位于src\kernel\integrator\path_state.h，

```
path.flag的一些定义
enum PathRayFlag : uint32_t {
  /* --------------------------------------------------------------------
   * Ray visibility.
   *
   * NOTE: Recalculated after a surface bounce.
   */

  PATH_RAY_CAMERA = (1U << 0U),
  PATH_RAY_REFLECT = (1U << 1U),
  PATH_RAY_TRANSMIT = (1U << 2U),
  PATH_RAY_DIFFUSE = (1U << 3U),
  PATH_RAY_GLOSSY = (1U << 4U),
  PATH_RAY_SINGULAR = (1U << 5U),
  PATH_RAY_TRANSPARENT = (1U << 6U),
  PATH_RAY_VOLUME_SCATTER = (1U << 7U),
  PATH_RAY_IMPORTANCE_BAKE = (1U << 8U),

  /* Shadow ray visibility. */
  PATH_RAY_SHADOW_OPAQUE = (1U << 9U),
  PATH_RAY_SHADOW_TRANSPARENT = (1U << 10U),
  PATH_RAY_SHADOW = (PATH_RAY_SHADOW_OPAQUE | PATH_RAY_SHADOW_TRANSPARENT),

  /* Subset of flags used for ray visibility for intersection.
   *
   * NOTE: SHADOW_CATCHER macros below assume there are no more than
   * 16 visibility bits. */
  PATH_RAY_ALL_VISIBILITY = ((1U << 11U) - 1U),
  ...
}

ccl_device_inline uint path_state_ray_visibility(ConstIntegratorState state)
{
  // 判断可见性的依据是 path.flag
  const uint32_t path_flag = INTEGRATOR_STATE(state, path, flag);
  // 这里是取出了flag的前10位，里面保存visibility信息
  uint32_t visibility = path_flag & PATH_RAY_ALL_VISIBILITY;

  /* For visibility, diffuse/glossy are for reflection only. */
  // tansmit光线不能是diffuse或glossy
  if (visibility & PATH_RAY_TRANSMIT) {
    visibility &= ~(PATH_RAY_DIFFUSE | PATH_RAY_GLOSSY);
  }

  /* todo: this is not supported as its own ray visibility yet. */
  if (path_flag & PATH_RAY_VOLUME_SCATTER) {
    visibility |= PATH_RAY_DIFFUSE;
  }

  visibility = SHADOW_CATCHER_PATH_VISIBILITY(path_flag, visibility);

  return visibility;
}

// SHADOW_CATCHER_VISIBILITY_SHIFT将visibility左移16位
#define SHADOW_CATCHER_PATH_VISIBILITY(path_flag, visibility) \
  (((path_flag)&PATH_RAY_SHADOW_CATCHER_PASS) ? SHADOW_CATCHER_VISIBILITY_SHIFT(visibility) : \
                                                (visibility))
```

获取visibility主要是获取PathRayFlag的前10位，里面包含visibility信息，在camera ray刚开始生成的时候，flag是这样被初始化的：

```
  INTEGRATOR_STATE_WRITE(state, path, flag) = PATH_RAY_CAMERA | PATH_RAY_MIS_SKIP |
                                              PATH_RAY_TRANSPARENT_BACKGROUND;
```

对于visibility信息来说，有效的是PATH_RAY_CAMERA，表明它是一条camera ray。

## GI近似

```
  /* Trick to use short AO rays to approximate indirect light at the end of the path. */
  if (path_state_ao_bounce(kg, state)) {
    ray.tmax = kernel_data.integrator.ao_bounces_distance;

    if (last_isect_object != OBJECT_NONE) {
      const float object_ao_distance = kernel_data_fetch(objects, last_isect_object).ao_distance;
      if (object_ao_distance != 0.0f) {
        ray.tmax = object_ao_distance;
      }
    }
  }
```

这里使用AO光线来近似间接光照，只在快要结束(弹射次数快结束)的path上使用，相当于牺牲一点效果来换取性能，对应Blender上的Fast GI Approximation特性。
![[Pasted image 20240812105307.png]]
path_state_ao_bounce源码位于src\kernel\integrator\path_state.h，
```
ccl_device_inline bool path_state_ao_bounce(KernelGlobals kg, ConstIntegratorState state)
{
  // 如果没有使用这个feature，直接返回false
  if (!kernel_data.integrator.ao_bounces) {
    return false;
  }
  // 这里是计算剩余的bounce此时是否大于ao_bounces，如果大于就启动GI近似
  // 为什么是大于，按照这个逻辑，应该是path开始阶段启动的GI近似
  const int bounce = INTEGRATOR_STATE(state, path, bounce) -
                     INTEGRATOR_STATE(state, path, transmission_bounce) -
                     (INTEGRATOR_STATE(state, path, glossy_bounce) > 0) + 1;
  return (bounce > kernel_data.integrator.ao_bounces);
}
```
如果启动了GI近似，首先就将ray.tmax 改为AO Distance，然后如果光线击中了物体，将ray.tmax改为物体的ao_distance（Obj中有ao_distance属性）。
## BVH求交
【Möller–Trumbore交叉算法】
bool hit = scene_intersect(kg, &ray, visibility, &isect);
求交的函数比较复杂，这里先不深入探索它的实现，简单了解这个函数的功能。首先，尽管传入了ray的地址，但求交算法并不会改变ray的数据，主要是填充isect，也就是相交的信息，Intersection结构体定义如下：

```
typedef struct Intersection {
  float t, u, v;
  int prim;
  int object;
  int type;
} Intersection;
```
t 是交点与射线原点的距离，射线的表达式时P + tD，有了t就可以计算交点的位置，
u, v表示三角形内的重心坐标（用来采样纹理？）
prim是交点的几何图元，
object是相交的物体，
type交点的图元类型，一般是PRIMITIVE_TRIANGLE。
## 光源的求交
```
  /* Light intersection for MIS. */
  if (kernel_data.integrator.use_light_mis && !integrator_intersect_skip_lights(kg, state)) {
    /* NOTE: if we make lights visible to camera rays, we'll need to initialize
     * these in the path_state_init. */
    const int last_type = INTEGRATOR_STATE(state, isect, type);
    hit = lights_intersect(
              kg, state, &ray, &isect, last_isect_prim, last_isect_object, last_type, path_flag) ||
          hit;
  }
```
要使用这个功能，首先需要打开光源的多重重要性采样功能，否则弹射的射线在场景求交时会忽略掉光源，只考虑物体的相交。
对应kernel_data.integrator.use_light_mis，这里还有一个!integrator_intersect_skip_lights的条件判断，它的源码如下：
```
ccl_device_forceinline bool integrator_intersect_skip_lights(KernelGlobals kg,
                                                             IntegratorState state)
{
  /* When direct lighting is disabled for baking, we skip light sampling in
   * integrate_surface_direct_light for the first bounce. Therefore, in order
   * for MIS to be consistent, we also need to skip evaluating lights here. */
  return (kernel_data.integrator.filter_closures & FILTER_CLOSURE_DIRECT_LIGHT) &&
         (INTEGRATOR_STATE(state, path, bounce) == 1);
}
```
烘培时禁用直接照明，跳过了第一次弹射，所以也需要跳过对光源的求交。

lights_intersect就是对射线与光源的求交，上面的BVH求交针对ray与场景中的物体，如果打开了MIS，还需要考虑ray与光源相交的情况，和上面的BVH求交一样，现在也不详细研究lights_intersect的具体实现，大概就是遍历场景中的灯光与当前ray依次求交，每一种光源的算法不同，最终相交的结果保存到isect，如果ray与光源相交的话，isect.type = PRIMITIVE_LAMP; isect->object = OBJECT_NONE。

经过BVH和光源的两次求交计算以后，将相交结果isect保存到state中。

## 跳转至下一kernel

```
  /* Setup up next kernel to be executed. */
  integrator_intersect_next_kernel<DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST>(
      kg, state, &isect, render_buffer, hit);
```

经过了求交计算，就可以根据相交的信息决定需要执行的下一kernel，integrator_intersect_next_kernel代码片段如下（省略了某些流程，比如体渲染时的kernel跳转）：
```
template<DeviceKernel current_kernel>
ccl_device_forceinline void integrator_intersect_next_kernel(
    KernelGlobals kg,
    IntegratorState state,
    ccl_private const Intersection *ccl_restrict isect,
    ccl_global float *ccl_restrict render_buffer,
    const bool hit)
{
#ifdef __VOLUME__
   ...
#endif
  if (hit) {
    /* Hit a surface, continue with light or surface kernel. */
    if (isect->type & PRIMITIVE_LAMP) {
      integrator_path_next(kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_LIGHT);
    }
    else {
      /* Hit a surface, continue with surface kernel unless terminated. */
      const int shader = intersection_get_shader(kg, isect);
      const int flags = kernel_data_fetch(shaders, shader).flags;

      if (!integrator_intersect_terminate(kg, state, flags)) {
        const int object_flags = intersection_get_object_flags(kg, isect);
        const bool use_caustics = kernel_data.integrator.use_caustics &&
                                  (object_flags & SD_OBJECT_CAUSTICS);
        const bool use_raytrace_kernel = (flags & SD_HAS_RAYTRACE);
        if (use_caustics) {
          integrator_path_next_sorted(
              kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_MNEE, shader);
        }
        else if (use_raytrace_kernel) {
          integrator_path_next_sorted(
              kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_RAYTRACE, shader);
        }
        else {
          integrator_path_next_sorted(
              kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE, shader);
        }

#ifdef __SHADOW_CATCHER__
   ...
#endif
      }
      else {
        integrator_path_terminate(kg, state, current_kernel);
      }
    }
  }
  else {
    /* Nothing hit, continue with background kernel. */
    if (integrator_intersect_skip_lights(kg, state)) {
      integrator_path_terminate(kg, state, current_kernel);
    }
    else {
      integrator_path_next(kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_BACKGROUND);
    }
  }
}
```
整理上面代码的逻辑，根据相交情况：

### 射线与物体相交

#### 与光源相交
下一个kernel为 DEVICE_KERNEL_INTEGRATOR_SHADE_LIGHT，进行光源的着色计算。
#### 与物体相交
先从相交的图元处取得shader,const int shader = intersection_get_shader(kg, isect);  
它的源码位于 src\kernel\bvh\util.h，

```
ccl_device_forceinline int intersection_get_shader(
    KernelGlobals kg, ccl_private const Intersection *ccl_restrict isect)
{
  return intersection_get_shader_from_isect_prim(kg, isect->prim, isect->type);
}

ccl_device_forceinline int intersection_get_shader_from_isect_prim(KernelGlobals kg,
                                                                   const int prim,
                                                                   const int isect_type)
{
  int shader = 0;

  if (isect_type & PRIMITIVE_TRIANGLE) {
    shader = kernel_data_fetch(tri_shader, prim);
  }
#ifdef __POINTCLOUD__
  ...
#endif
#ifdef __HAIR__
  ...
#endif
  return shader & SHADER_MASK;
}

typedef enum ShaderFlag {
  SHADER_SMOOTH_NORMAL = (1 << 31),
  SHADER_CAST_SHADOW = (1 << 30),
  SHADER_AREA_LIGHT = (1 << 29),
  SHADER_USE_MIS = (1 << 28),
  SHADER_EXCLUDE_DIFFUSE = (1 << 27),
  SHADER_EXCLUDE_GLOSSY = (1 << 26),
  SHADER_EXCLUDE_TRANSMIT = (1 << 25),
  SHADER_EXCLUDE_CAMERA = (1 << 24),
  SHADER_EXCLUDE_SCATTER = (1 << 23),
  SHADER_EXCLUDE_SHADOW_CATCHER = (1 << 22),
  SHADER_EXCLUDE_ANY = (SHADER_EXCLUDE_DIFFUSE | SHADER_EXCLUDE_GLOSSY | SHADER_EXCLUDE_TRANSMIT |
                        SHADER_EXCLUDE_CAMERA | SHADER_EXCLUDE_SCATTER |
                        SHADER_EXCLUDE_SHADOW_CATCHER),

  SHADER_MASK = ~(SHADER_SMOOTH_NORMAL | SHADER_CAST_SHADOW | SHADER_AREA_LIGHT | SHADER_USE_MIS |
                  SHADER_EXCLUDE_ANY)
} ShaderFlag;
```
shader是一个int类型的值，是一个bitmap，最后和SHADER_MASK进行&运算表示去除SHADER_SMOOTH_NORMAL，SHADER_CAST_SHADOW 等标志位。
  
取出shder之后，又继续取shader的flags，const int flags = kernel_data_fetch(shaders, shader).flags; 其中shaders的类型为KernelShader ，它的定义如下：
```
typedef struct KernelShader {
  float constant_emission[3];
  float cryptomatte_id;
  int flags;
  int pass_id;
  int pad2, pad3;
} KernelShader;
```
KernelShader用来保存kernel中使用的shader，这里就是从所有shader中取出了当前图元的shader的flag。

有的光线即使击中了场景中的物体也不需要进行shader计算，通过此flag可以判断当前光线是否应该结束，integrator_intersect_terminate函数用来进行此判断，它的源码：
```cpp
ccl_device_forceinline bool integrator_intersect_terminate(KernelGlobals kg,
                                                           IntegratorState state,
                                                           const int shader_flags)
{
  /* Optional AO bounce termination.
   * We continue evaluating emissive/transparent surfaces and volumes, similar
   * to direct lighting. Only if we know there are none can we terminate the
   * path immediately. */
  // 如果当前ray是AO ray的话，除非当前shader有TRANSPARENT_SHADOW或HAS_EMISSION
  // 或者!integrator_state_volume_stack_is_empty（应该是关于体渲染的）
  // 否则此ray应该中止
  // AO光线本来就不应该计算shader，除非遇到透明和自发光材质，它们不会造成遮挡
  if (path_state_ao_bounce(kg, state)) {
    if (shader_flags & (SD_HAS_TRANSPARENT_SHADOW | SD_HAS_EMISSION)) {
      INTEGRATOR_STATE_WRITE(state, path, flag) |= PATH_RAY_TERMINATE_AFTER_TRANSPARENT;
    }
    else if (!integrator_state_volume_stack_is_empty(kg, state)) {
      INTEGRATOR_STATE_WRITE(state, path, flag) |= PATH_RAY_TERMINATE_AFTER_VOLUME;
    }
    else {
      return true;
    }
  }

  /* Load random number state. */
  // 读取的是path.rng_hash, path.rng_offset, path.sample三个属性
  RNGState rng_state;
  path_state_rng_load(state, &rng_state);

  /* We perform path termination in this kernel to avoid launching shade_surface
   * and evaluating the shader when not needed. Only for emission and transparent
   * surfaces in front of emission do we need to evaluate the shader, since we
   * perform MIS as part of indirect rays. */
  // 这里开始俄罗斯轮盘赌，不过会保证弹射次数不小于用户配置的min_bounce
  const uint32_t path_flag = INTEGRATOR_STATE(state, path, flag);
  const float continuation_probability = path_state_continuation_probability(kg, state, path_flag);
  INTEGRATOR_STATE_WRITE(state, path, continuation_probability) = continuation_probability;

  guiding_record_continuation_probability(kg, state, continuation_probability);

  if (continuation_probability != 1.0f) {
    // 当ray继续弹射的概率小于1时，需要考虑自发光物体或体渲染时的情况
    const float terminate = path_state_rng_1D(kg, &rng_state, PRNG_TERMINATE);

    if (continuation_probability == 0.0f || terminate >= continuation_probability) {
      if (shader_flags & SD_HAS_EMISSION) {
        /* Mark path to be terminated right after shader evaluation on the surface. */
        INTEGRATOR_STATE_WRITE(state, path, flag) |= PATH_RAY_TERMINATE_ON_NEXT_SURFACE;
      }
      else if (!integrator_state_volume_stack_is_empty(kg, state)) {
        /* TODO: only do this for emissive volumes. */
        INTEGRATOR_STATE_WRITE(state, path, flag) |= PATH_RAY_TERMINATE_IN_NEXT_VOLUME;
      }
      else {
        return true;
      }
    }
  }

  return false;
}
```
从上面的代码可以看出，判断一条光线结束最主要的手段就是[[蒙特卡洛积分和采样]]中接介绍的俄罗斯轮盘方法，不过这里并使用的并不是终止概率（termination probability），而是继续概率（continuation probability），不过其思想是一样的。其中求continuation_probability时使用到了函数path_state_continuation_probability，它的源码片段：
```cpp
ccl_device_inline float path_state_continuation_probability(KernelGlobals kg,
                                                            ConstIntegratorState state,
                                                            const uint32_t path_flag)
{
  if (path_flag & PATH_RAY_TRANSPARENT) {
    const uint32_t transparent_bounce = INTEGRATOR_STATE(state, path, transparent_bounce);
    /* Do at least specified number of bounces without RR. */
    if (transparent_bounce <= kernel_data.integrator.transparent_min_bounce) {
      return 1.0f;
    }
  }
  else {
    const uint32_t bounce = INTEGRATOR_STATE(state, path, bounce);
    /* Do at least specified number of bounces without RR. */
    if (bounce <= kernel_data.integrator.min_bounce) {
      return 1.0f;
    }
  }

  /* Probabilistic termination: use sqrt() to roughly match typical view
   * transform and do path termination a bit later on average. */
  Spectrum throughput = INTEGRATOR_STATE(state, path, throughput);
#if defined(__PATH_GUIDING__) && PATH_GUIDING_LEVEL >= 4
  ...
#endif
  return min(sqrtf(reduce_max(fabs(throughput))), 1.0f);
}
```
首先，如果光线的弹射次数还没有达到配置的最小弹射次数时，continuation_probability = 1，此光线必然不会被俄罗斯轮盘终止。如果是其他情况，则通过throughput来计算continuation_probability，具体的方法是取throughput的3个分量的最大值再开平方，并保证它不会大于1。在一条路径初始化时，它的throughput=(1.0, 1.0, 1.0)，所以最开始从相机发射的光线不会在这里终止，后面随着throughput的降低，路径终止的概率会越来越大。
获取到continuation_probability后，如果此概率小于1，即可能出现终止的情况，那么生成一个0到1之间的随机数terminate，如果terminate >= continuation_probability，即随机数落到了continuation_probability之外，那么这条路径就需要终止。
终止时有两种特殊情况，光线击中的是自发光物体或体渲染物体，那么它不会立即终止，而是将路径状态置为PATH_RAY_TERMINATE_ON_NEXT_SURFACE或PATH_RAY_TERMINATE_IN_NEXT_VOLUME，标记它们会在后面正确的时机终止。
如果光线中止，调用下面这个函数终止光线计算：
```cpp
integrator_path_terminate(kg, state, current_kernel);
```
它的源码位于src\kernel\integrator\state_flow.h，
```cpp
ccl_device_forceinline void integrator_path_terminate(KernelGlobals kg,
                                                      IntegratorState state,
                                                      const DeviceKernel current_kernel)
{
  INTEGRATOR_STATE_WRITE(state, path, queued_kernel) = 0;
  (void)current_kernel;
}
```

将queued_kernel置为0，表示此光线无需在进行后续的kernel计算。

假设光线未终止，那么下一步就是根据光线与物体相交的信息和渲染设置开始对像素进行着色，对应这段代码
```
        const int object_flags = intersection_get_object_flags(kg, isect);
        const bool use_caustics = kernel_data.integrator.use_caustics &&
                                  (object_flags & SD_OBJECT_CAUSTICS);
        const bool use_raytrace_kernel = (flags & SD_HAS_RAYTRACE);
        if (use_caustics) {
          integrator_path_next_sorted(
              kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_MNEE, shader);
        }
        else if (use_raytrace_kernel) {
          integrator_path_next_sorted(
              kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_RAYTRACE, shader);
        }
        else {
          integrator_path_next_sorted(
              kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE, shader);
        }
```
如果使用了焦散，那么下一个kernel就是DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_MNEE，如果shader中使用了raytrace_kernel（现在还不知道哪些kernel是raytrace_kernel），那么下一个kernel就是DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE_RAYTRACE，如果不是以上两种情况，那么下一个kernel就是DEVICE_KERNEL_INTEGRATOR_SHADE_SURFACE，这是最常用的着色kernel。
### 射线与物体未相交
```
    /* Nothing hit, continue with background kernel. */
    if (integrator_intersect_skip_lights(kg, state)) {
      integrator_path_terminate(kg, state, current_kernel);
    }
    else {
      integrator_path_next(kg, state, current_kernel, DEVICE_KERNEL_INTEGRATOR_SHADE_BACKGROUND);
    }
```
如果光线被跳过，直接终止此光线，否则进入背景计算的kernel DEVICE_KERNEL_INTEGRATOR_SHADE_BACKGROUND。
## 总结
intergrator_intersect_closest用于进行光线和场景的求交计算，并根据相交的结果决定需要执行的下一个kernel。在这个过程中，最重要的就是求得相交信息Intersection，它将用于后续的光照计算。除此之外，这里还根据不同的情况决定了一条光线需不需要继续传播。