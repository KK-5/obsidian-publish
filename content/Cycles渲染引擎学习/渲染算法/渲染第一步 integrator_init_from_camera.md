在对渲染流程的分析中可以知道，渲染的第一步是从integrator_init_from_camera或integrator_init_from_bake开始的，对应摄像机渲染图片和烘培操作。这里就从integrator_init_from_camera开始，逐步研究cycles渲染的算法。

integrator_init_from_camera源码：
```cpp
/* Return false to indicate that this pixel is finished.
 * Used by CPU implementation to not attempt to sample pixel for multiple samples once its known
 * that the pixel did converge. */
ccl_device bool integrator_init_from_camera(KernelGlobals kg,
                                            IntegratorState state,
                                            ccl_global const KernelWorkTile *ccl_restrict tile,
                                            ccl_global float *render_buffer,
                                            const int x,
                                            const int y,
                                            const int scheduled_sample)
{
  // 在GPU中KernelGlobals不使用，所以忽略kg的任何操作
  PROFILING_INIT(kg, PROFILING_RAY_SETUP);

  /* Initialize path state to give basic buffer access and allow early outputs. */
  path_state_init(state, tile, x, y);

  /* Check whether the pixel has converged and should not be sampled anymore. */
  if (!film_need_sample_pixel(kg, state, render_buffer)) {
    return false;
  }

  /* Count the sample and get an effective sample for this pixel.
   *
   * This logic allows to both count actual number of samples per pixel, and to add samples to this
   * pixel after it was converged and samples were added somewhere else (in which case the
   * `scheduled_sample` will be different from actual number of samples in this pixel). */
  const int sample = film_write_sample(
      kg, state, render_buffer, scheduled_sample, tile->sample_offset);

  /* Initialize random number seed for path. */
  const uint rng_hash = path_rng_hash_init(kg, sample, x, y);

  {
    /* Generate camera ray. */
    Ray ray;
    integrate_camera_sample(kg, sample, x, y, rng_hash, &ray);
    if (ray.tmax == 0.0f) {
      return true;
    }

    /* Write camera ray to state. */
    integrator_state_write_ray(state, &ray);
  }

  /* Initialize path state for path integration. */
  path_state_init_integrator(kg, state, sample, rng_hash);

  /* Continue with intersect_closest kernel, optionally initializing volume
   * stack before that if the camera may be inside a volume. */
  if (kernel_data.cam.is_inside_volume) {
    integrator_path_init(kg, state, DEVICE_KERNEL_INTEGRATOR_INTERSECT_VOLUME_STACK);
  }
  else {
    integrator_path_init(kg, state, DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST);
  }

  return true;
}
```

从上面这个函数可以看到，integrator_init_from_camera的主要功能是从相机生成camera ray，然后将其保存在state里面，然后切换到下一个kernel。如果此像素已经计算完毕，返回false。
逐个分析算法的每个步骤：
## path_state_init

根据注释描述，这个步骤用于初始化路径状态以提供基本缓冲区访问权限并允许早期的输出。用于渲染期间的结果输出，它的源码位于src\kernel\integrator\path_state.h，

```
/* Minimalistic initialization of the path state, which is needed for early outputs in the
 * integrator initialization to work. */
ccl_device_inline void path_state_init(IntegratorState state,
                                       ccl_global const KernelWorkTile *ccl_restrict tile,
                                       const int x,
                                       const int y)
{
  const uint render_pixel_index = (uint)tile->offset + x + y * tile->stride;
  // 初始化 render_pixel_index
  INTEGRATOR_STATE_WRITE(state, path, render_pixel_index) = render_pixel_index;

  path_state_init_queues(state);
}

/* Initialize queues, so that the this path is considered terminated.
 * Used for early outputs in the camera ray initialization, as well as initialization of split
 * states for shadow catcher. */
ccl_device_inline void path_state_init_queues(IntegratorState state)
{
  // queued_kernel是一个bitmap，每一个bit表示下一个需要执行的kernel，这里将它初始化为0，
  // 表示还没有下一个需要执行的kernel
  INTEGRATOR_STATE_WRITE(state, path, queued_kernel) = 0;
#ifndef __KERNEL_GPU__
  INTEGRATOR_STATE_WRITE(&state->shadow, shadow_path, queued_kernel) = 0;
  INTEGRATOR_STATE_WRITE(&state->ao, shadow_path, queued_kernel) = 0;
#endif
}
```

这个步骤只是执行了初始化操作，既然描述上说是为了提供基本缓冲区访问权限，是否表示这些初始化对于渲染并不是必要的。

## film_need_sample_pixel

注释说这个步骤是检查此像素是否已收敛，可以不用再进行采样，源码位于src\kernel\film\adaptive_sampling.h，

```
/* Check whether the pixel has converged and should not be sampled anymore. */

ccl_device_forceinline bool film_need_sample_pixel(KernelGlobals kg,
                                                   ConstIntegratorState state,
                                                   ccl_global float *render_buffer)
{
  if (kernel_data.film.pass_adaptive_aux_buffer == PASS_UNUSED) {
    return true;
  }

  const uint32_t render_pixel_index = INTEGRATOR_STATE(state, path, render_pixel_index);
  const uint64_t render_buffer_offset = (uint64_t)render_pixel_index *
                                        kernel_data.film.pass_stride;
  ccl_global float *buffer = render_buffer + render_buffer_offset;

  const uint aux_w_offset = kernel_data.film.pass_adaptive_aux_buffer + 3;
  return buffer[aux_w_offset] == 0.0f;
}
```

这个步骤有点难以理解，看起来像素是否已经收敛是通过pass_adaptive_aux_buffer的值来判断的，根据一些代码注释，pass_adaptive_aux_buffer属于自适应采样的属性，自适应采样的算法与一篇名为“A hierarchical automatic stopping condition for Monte Carlo global illumination”的文章有关。后面再详细研究它的算法。

## film_write_sample

对采样进行计数并获取该像素的有效样本，也可以想该像素添加样本，

源码位于src\kernel\film\light_passes.h

```
/* --------------------------------------------------------------------
 * Adaptive sampling.
 */

ccl_device_inline int film_write_sample(KernelGlobals kg,
                                        ConstIntegratorState state,
                                        ccl_global float *ccl_restrict render_buffer,
                                        int sample,
                                        int sample_offset)
{
  if (kernel_data.film.pass_sample_count == PASS_UNUSED) {
    return sample;
  }

  ccl_global float *buffer = film_pass_pixel_render_buffer(kg, state, render_buffer);

  return atomic_fetch_and_add_uint32(
             (ccl_global uint *)(buffer) + kernel_data.film.pass_sample_count, 1) +
         sample_offset;
}
```

传入的sample参数是理论采样计数，返回实际的采样计数。

这个阶段也属于自适应采样算法，暂时不深入研究。

## path_rng_hash_init

生成随机种子，源码位于src\kernel\sample\pattern.h，

```
ccl_device_inline uint path_rng_hash_init(KernelGlobals kg,
                                          const int sample,
                                          const int x,
                                          const int y)
{
  const uint rng_hash = hash_iqnt2d(x, y) ^ kernel_data.integrator.seed;

#ifdef __DEBUG_CORRELATION__
  srand48(rng_hash + sample);
#else
  (void)sample;
#endif

  return rng_hash;
}
/**
 * 2D hash recommended from "Hash Functions for GPU Rendering" JCGT Vol. 9, No. 3, 2020
 * See https://www.shadertoy.com/view/4tXyWN and https://www.shadertoy.com/view/XlGcRh
 * http://www.jcgt.org/published/0009/03/02/paper.pdf
 */
ccl_device_inline uint hash_iqnt2d(const uint x, const uint y)
{
  const uint qx = 1103515245U * ((x >> 1U) ^ (y));
  const uint qy = 1103515245U * ((y >> 1U) ^ (x));
  const uint n = 1103515245U * ((qx) ^ (qy >> 3U));

  return n;
}
```

这个随机数用于camera ray的生成，随机数的产生规则在注释中的论文里。

## Generate & Write camera ray

camera ray的生成和写入，对应这段代码：

```
  {
    /* Generate camera ray. */
    Ray ray;
    integrate_camera_sample(kg, sample, x, y, rng_hash, &ray);
    if (ray.tmax == 0.0f) {
      return true;
    }

    /* Write camera ray to state. */
    integrator_state_write_ray(state, &ray);
  }
```

### integrate_camera_sample

integrate_camera_sample与integrator_init_from_camera定义在同一文件：

```
ccl_device_inline void integrate_camera_sample(KernelGlobals kg,
                                               const int sample,
                                               const int x,
                                               const int y,
                                               const uint rng_hash,
                                               ccl_private Ray *ray)
{
  /* Filter sampling. */
  // 这里生成二维的随机数rand_filter，根据之前生成的rng_hash来生成
  // 使用的生成随机数的算法是sobol_burley和tabulated_sobol
  // sobol_burle多用于在渲染时生成均匀分布的采样点
  // tabulated_sobol通过预计算的方式提高采样点生成速率
  // 这里暂时了解一下，只需要知道生成了float2的随机数
  const float2 rand_filter = (sample == 0) ? make_float2(0.5f, 0.5f) :
                                             path_rng_2D(kg, rng_hash, sample, PRNG_FILTER);

  /* Motion blur (time) and depth of field (lens) sampling. (time, lens_x, lens_y) */
  // 只有运动模糊和景深需要rand_time_lens，这里就把它视作 float3(0, 0, 0)
  const float3 rand_time_lens = (kernel_data.cam.shuttertime != -1.0f ||
                                 kernel_data.cam.aperturesize > 0.0f) ?
                                    path_rng_3D(kg, rng_hash, sample, PRNG_LENS_TIME) :
                                    zero_float3();

  /* We use x for time and y,z for lens because in practice with Sobol
   * sampling this seems to give better convergence when an object is
   * both motion blurred and out of focus, without significantly harming
   * convergence for focal blur alone.  This is a little surprising,
   * because one would expect using x,y for lens (the 2d part) would be
   * best, since x,y are the best stratified.  Since it's not entirely
   * clear why this is, this is probably worth revisiting at some point
   * to investigate further. */
  const float rand_time = rand_time_lens.x;
  const float2 rand_lens = make_float2(rand_time_lens.y, rand_time_lens.z);

  /* Generate camera ray. */
  camera_sample(kg, x, y, rand_filter, rand_time, rand_lens, ray);
}
```

#### camera_sample
这是生成camera ray的函数，通过像素x, y和随机数生成camera ray，源码位于src\kernel\camera\camera.h
```
ccl_device_inline void camera_sample(KernelGlobals kg,
                                     int x,
                                     int y,
                                     const float2 filter_uv,
                                     const float time,
                                     const float2 lens_uv,
                                     ccl_private Ray *ray)
{
  /* pixel filter */
  // 这里的filter_table_offset是一个查找表，暂不清楚是什么含义
  // 这里从x和y生成raster，就是通过将x和y偏移一个从filter_table_offset中找到的值
  // 比如x = 30, y = 40，找到的raster也许是(30.2, 40.3)？
  int filter_table_offset = kernel_data.tables.filter_table_offset;
  const float2 raster = make_float2(
      x + lookup_table_read(kg, filter_uv.x, filter_table_offset, FILTER_TABLE_SIZE),
      y + lookup_table_read(kg, filter_uv.y, filter_table_offset, FILTER_TABLE_SIZE));

  // 下面这段if-else语句是为了处理运动模糊和景深，可忽略
  /* motion blur */
  if (kernel_data.cam.shuttertime == -1.0f) {
    ...
  }
  else {
    ...
  }

  /* sample */
  if (kernel_data.cam.type == CAMERA_PERSPECTIVE) {
    camera_sample_perspective(kg, raster, lens_uv, ray);
  }
  else if (kernel_data.cam.type == CAMERA_ORTHOGRAPHIC) {
    camera_sample_orthographic(kg, raster, lens_uv, ray);
  }
  else {
    ccl_global const DecomposedTransform *cam_motion = kernel_data_array(camera_motion);
    camera_sample_panorama(&kernel_data.cam, cam_motion, raster, lens_uv, ray);
  }
}
```

通过x和y生成好raster后，再生成根据不同的相机种类生成camera ray，通常都是使用透视投影的相机，所以这里看camera_sample_perspective函数，位于src\kernel\camera\camera.h，这里展示核心的代码，

```
ccl_device void camera_sample_perspective(KernelGlobals kg,
                                          const float2 raster_xy,
                                          const float2 rand_lens,
                                          ccl_private Ray *ray)
{
  /* create ray form raster position */
  // 将屏幕上的点变换到视图空间
  ProjectionTransform rastertocamera = kernel_data.cam.rastertocamera;
  const float3 raster = float2_to_float3(raster_xy);
  float3 Pcamera = transform_perspective(&rastertocamera, raster);
  
  // 摄像机是静止的，这里忽略
  if (kernel_data.cam.have_perspective_motion) {
    ...
  }
  // 在视图空间中，原点为相机位置，射线方向从相机指向屏幕上的点，所以D = Pcamera
  float3 P = zero_float3();
  float3 D = Pcamera;

  // 没有使用景深，这里忽略
  /* modify ray for depth of field */
  float aperturesize = kernel_data.cam.aperturesize;

  if (aperturesize > 0.0f) {
     ...
  }

  /* transform ray from camera to world */
  Transform cameratoworld = kernel_data.cam.cameratoworld;

  // 摄像机是静止的，这里忽略
  if (kernel_data.cam.num_motion_steps) {
     ...
  }
  // 把P和D变换到世界空间
  P = transform_point(&cameratoworld, P);
  D = normalize(transform_direction(&cameratoworld, D));
  // 普通的相机，use_stereo = false
  bool use_stereo = kernel_data.cam.interocular_offset != 0.0f;
  if (!use_stereo) {
    /* No stereo */
    ray->P = P;
    ray->D = D;

#ifdef __RAY_DIFFERENTIALS__
    float3 Dcenter = transform_direction(&cameratoworld, Pcamera);
    float3 Dcenter_normalized = normalize(Dcenter);

    /* TODO: can this be optimized to give compact differentials directly? */
    // 计算 dP 和 dD
    // dP = 0
    ray->dP = differential_zero_compact();
    differential3 dD;
    // dD的每个分量计算都是 Dcenter + cam.d - Dcenter_normalized
    // differential_make_compact = 0.5f * (len(dD.dx) + len(dD.dy))
    dD.dx = normalize(Dcenter + float4_to_float3(kernel_data.cam.dx)) - Dcenter_normalized;
    dD.dy = normalize(Dcenter + float4_to_float3(kernel_data.cam.dy)) - Dcenter_normalized;
    ray->dD = differential_make_compact(dD);
#endif
  }
  else {
    /* Spherical stereo */
    ...
  }

  /* clipping */
  // z_inv 表示raster变换到视图空间后的深度的倒数
  // 为什么要计算z_inv，并用它与nearclip和cliplength相乘？
  float z_inv = 1.0f / normalize(Pcamera).z;
  float nearclip = kernel_data.cam.nearclip * z_inv;
  ray->P += nearclip * ray->D;
  ray->dP += nearclip * ray->dD;
  ray->tmin = 0.0f;
  // cliplength = farclip - nearclip
  ray->tmax = kernel_data.cam.cliplength * z_inv;
}
```

将运动模糊和景深忽略后，这里假设只是普通的透视渲染。光线生成的流程很清晰，首先将raster，即屏幕上的点变换到视图空间，由于第一条光线一定是从相机发出的（视图空间下相机位置为（0, 0, 0）），所以光线的起点就是原点，方向就是原点指向该采样点的方向。然后再将光线的位置和方向变换到世界空间中，并根据需要计算dD，dP，还有光线的起始和结束点。

有两个地方比较复杂：

1. dP和dD的计算  
    dP和dD表示光线原点和方向的偏微分，可以表示光线在场景中的变化情况，用于在纹理采样时提高采样精度。  
    关于光线微分，见 [https://dezeming.top/wp-content/uploads/2022/07/PBRT系列14-代码实战-光线微分与纹理.pdf](https://dezeming.top/wp-content/uploads/2022/07/PBRT%E7%B3%BB%E5%88%9714-%E4%BB%A3%E7%A0%81%E5%AE%9E%E6%88%98-%E5%85%89%E7%BA%BF%E5%BE%AE%E5%88%86%E4%B8%8E%E7%BA%B9%E7%90%86.pdf)
    
2. clipping  
    在这个阶段，限制了光线的最近和最远距离，保证光照只会存在相机视锥体内。  
    为什么要使用z_inv，然后在计算nearclip和cliplength时乘上它？（校正？）
     #Doubtful 
    

## path_state_init_integrator

为path integration初始化path的状态，实际就是针对该state的path属性进行初始化，源码位于src\kernel\integrator\path_state.h

```
/* Initialize the rest of the path state needed to continue the path integration. */
ccl_device_inline void path_state_init_integrator(KernelGlobals kg,
                                                  IntegratorState state,
                                                  const int sample,
                                                  const uint rng_hash)
{
  INTEGRATOR_STATE_WRITE(state, path, sample) = sample;
  INTEGRATOR_STATE_WRITE(state, path, bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, diffuse_bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, glossy_bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, transmission_bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, transparent_bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, volume_bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, volume_bounds_bounce) = 0;
  INTEGRATOR_STATE_WRITE(state, path, rng_hash) = rng_hash;
  INTEGRATOR_STATE_WRITE(state, path, rng_offset) = PRNG_BOUNCE_NUM;
  INTEGRATOR_STATE_WRITE(state, path, flag) = PATH_RAY_CAMERA | PATH_RAY_MIS_SKIP |
                                              PATH_RAY_TRANSPARENT_BACKGROUND;
  INTEGRATOR_STATE_WRITE(state, path, mis_ray_pdf) = 0.0f;
  INTEGRATOR_STATE_WRITE(state, path, min_ray_pdf) = FLT_MAX;
  INTEGRATOR_STATE_WRITE(state, path, continuation_probability) = 1.0f;
  INTEGRATOR_STATE_WRITE(state, path, throughput) = one_spectrum();
  ...
}
```

都是将其中path的属性进行初始化。

## integrator_path_init

对state中的path进行初始化，指定下一个执行的kernel，

```
  /* Continue with intersect_closest kernel, optionally initializing volume
   * stack before that if the camera may be inside a volume. */
  if (kernel_data.cam.is_inside_volume) {
    integrator_path_init(kg, state, DEVICE_KERNEL_INTEGRATOR_INTERSECT_VOLUME_STACK);
  }
  else {
    integrator_path_init(kg, state, DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST);
  }
```

这里根据相机是否在volume中分为两个kernel，大多数情况下相机不会位于volume中，下一个kernel通常是DEVICE_KERNEL_INTEGRATOR_INTERSECT_CLOSEST，即计算光线与场景的交点。

integrator_path_init函数位于src\kernel\integrator\state_flow.h，

```
ccl_device_forceinline void integrator_path_init(KernelGlobals kg,
                                                 IntegratorState state,
                                                 const DeviceKernel next_kernel)
{
  INTEGRATOR_STATE_WRITE(state, path, queued_kernel) = next_kernel;
}
```

path中的queued_kernel是一个bitmap，保存了后续需要执行的kernel。

## 总结

integrator_init_from_camera是渲染时运行的第一个kernel，它的主要工作是生成camera ray，并完成一些必要的初始化，然后根据相机是否在volume中决定要执行的后续kernel。