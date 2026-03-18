[[反射模型]]和[[常用的反射模型]]中已经将渲染中PBR的机制和实现方案了解得差不多了。然而在应用中，这套PBR方案应用并不广泛。观察可以发现，PBR中的概念和参数繁多且复杂，并且并不灵活，需要考虑一个物体是导体还是绝缘体，需要考虑物体进行的是镜面反射还是漫反射，亦或是有粗糙度的微平面反射，并且这些不同类型的BxDF实现方案差异巨大。
disney BRDF是目前应用最为广泛的反射模型，几乎所有渲染器都支持此模型，或者以此为基础进行扩展。
顾名思义disney BRDF是BRDF模型，所以主要考虑的是光线的反射，对于光线的透射使用了一些近似方案来模拟，它也非常适合用于实时渲染领域。
# disney BRDF的理念
disney BRDF核心理念如下（直接贴了毛星云大佬的总结，disney BRDF原文章里也有介绍）：
- 应使用直观的参数，而不是物理类的晦涩参数。  
- 参数应尽可能少。  
- 参数在其合理范围内应该为0到1。  
- 允许参数在有意义时超出正常的合理范围。  
- 所有参数组合应尽可能健壮和合理。
其注重实用性大于物体正确性，对美术更加友好。

下面是disney BRDF的参数（使用metallic-roughness工作流）：
- baseColor（固有色）：表面颜色，通常由纹理贴图提供。
- subsurface（次表面）：使用次表面近似控制漫反射形状。
- metallic（金属度）：金属（0 = 电介质，1 =金属）。这是两种不同模型之间的线性混合。金属模型没有漫反射成分，并且还具有等于基础色的着色入射镜面反射。
- specular（镜面反射强度）：入射镜面反射量。用于取代折射率。
- specularTint（镜面反射颜色）：对美术控制的让步，用于对基础色（basecolor）的入射镜面反射进行颜色控制。掠射镜面反射仍然是非彩色的。
- roughness（粗糙度）：表面粗糙度，控制漫反射和镜面反射。
- anisotropic（各向异性强度）：各向异性程度。用于控制镜面反射高光的纵横比。（0 =各向同性，1 =最大各向异性。）
- sheen（光泽度）：一种额外的掠射分量（grazing component），主要用于布料。
- sheenTint（光泽颜色）：对sheen（光泽度）的颜色控制，也是一个美术参数。
- clearcoat（清漆强度）：有特殊用途的第二个镜面波瓣（specular lobe）。
- clearcoatGloss（清漆光泽度）：控制透明涂层光泽度，0 = “缎面（satin）”外观，1 = “光泽（gloss）”外观。
下面是官方提供的各参数效果示意图：
![[Pasted image 20250806145238.png]]

# disney BRDF的实现原理
## 总览
disney BRDF由多种类型的BRDF混合而来，其实现原理图如下：
![[Pasted image 20250806163033.png]]
总体可以将其概括为DielectricBRDF与ConductorBRDF的混合，使用参数metallic控制混合比例。
对于DielectricBRDF，它由DiffuseBRDF和SpecularBRDF混合而来，依然使用[[反射模型#Torrance–Sparrow BRDF]]的思想，即
$$
f_r(\omega_i,\omega_o)=f_{diffuse}+\frac{D(\omega_h)F(\omega_h,\omega_o)G(\omega_i, \omega_o)}{4\cos\theta_i\cos\theta_o}
$$
对于ConductorBRDF，它只包含specular部分，因为金属材质吸收光线能力强，不再发生漫反射和次表面散射。
为了方便，下面使用金属和非金属来表示导体和绝缘体。

下面是disney BRDF的官方实现示例（[disney_brdf](https://github.com/wdas/brdf/blob/main/src/brdfs/disney.brdf)）:
```
# variables go here...
# [type] [name] [min val] [max val] [default val]
::begin parameters
color baseColor .82 .67 .16
float metallic 0 1 0
float subsurface 0 1 0
float specular 0 1 .5
float roughness 0 1 .5
float specularTint 0 1 0
float anisotropic 0 1 0
float sheen 0 1 0
float sheenTint 0 1 .5
float clearcoat 0 1 0
float clearcoatGloss 0 1 1
::end parameters


::begin shader

const float PI = 3.14159265358979323846;

float sqr(float x) { return x*x; }

float SchlickFresnel(float u)
{
    float m = clamp(1-u, 0, 1);
    float m2 = m*m;
    return m2*m2*m; // pow(m,5)
}

float GTR1(float NdotH, float a)
{
    if (a >= 1) return 1/PI;
    float a2 = a*a;
    float t = 1 + (a2-1)*NdotH*NdotH;
    return (a2-1) / (PI*log(a2)*t);
}

float GTR2(float NdotH, float a)
{
    float a2 = a*a;
    float t = 1 + (a2-1)*NdotH*NdotH;
    return a2 / (PI * t*t);
}

float GTR2_aniso(float NdotH, float HdotX, float HdotY, float ax, float ay)
{
    return 1 / (PI * ax*ay * sqr( sqr(HdotX/ax) + sqr(HdotY/ay) + NdotH*NdotH ));
}

float smithG_GGX(float NdotV, float alphaG)
{
    float a = alphaG*alphaG;
    float b = NdotV*NdotV;
    return 1 / (NdotV + sqrt(a + b - a*b));
}

float smithG_GGX_aniso(float NdotV, float VdotX, float VdotY, float ax, float ay)
{
    return 1 / (NdotV + sqrt( sqr(VdotX*ax) + sqr(VdotY*ay) + sqr(NdotV) ));
}

vec3 mon2lin(vec3 x)
{
    return vec3(pow(x[0], 2.2), pow(x[1], 2.2), pow(x[2], 2.2));
}


vec3 BRDF( vec3 L, vec3 V, vec3 N, vec3 X, vec3 Y )
{
    float NdotL = dot(N,L);
    float NdotV = dot(N,V);
    if (NdotL < 0 || NdotV < 0) return vec3(0);

    vec3 H = normalize(L+V);
    float NdotH = dot(N,H);
    float LdotH = dot(L,H);

    vec3 Cdlin = mon2lin(baseColor);
    float Cdlum = .3*Cdlin[0] + .6*Cdlin[1]  + .1*Cdlin[2]; // luminance approx.

    vec3 Ctint = Cdlum > 0 ? Cdlin/Cdlum : vec3(1); // normalize lum. to isolate hue+sat
    vec3 Cspec0 = mix(specular*.08*mix(vec3(1), Ctint, specularTint), Cdlin, metallic);
    vec3 Csheen = mix(vec3(1), Ctint, sheenTint);

    // Diffuse fresnel - go from 1 at normal incidence to .5 at grazing
    // and mix in diffuse retro-reflection based on roughness
    float FL = SchlickFresnel(NdotL), FV = SchlickFresnel(NdotV);
    float Fd90 = 0.5 + 2 * LdotH*LdotH * roughness;
    float Fd = mix(1.0, Fd90, FL) * mix(1.0, Fd90, FV);

    // Based on Hanrahan-Krueger brdf approximation of isotropic bssrdf
    // 1.25 scale is used to (roughly) preserve albedo
    // Fss90 used to "flatten" retroreflection based on roughness
    float Fss90 = LdotH*LdotH*roughness;
    float Fss = mix(1.0, Fss90, FL) * mix(1.0, Fss90, FV);
    float ss = 1.25 * (Fss * (1 / (NdotL + NdotV) - .5) + .5);

    // specular
    float aspect = sqrt(1-anisotropic*.9);
    float ax = max(.001, sqr(roughness)/aspect);
    float ay = max(.001, sqr(roughness)*aspect);
    float Ds = GTR2_aniso(NdotH, dot(H, X), dot(H, Y), ax, ay);
    float FH = SchlickFresnel(LdotH);
    vec3 Fs = mix(Cspec0, vec3(1), FH);
    float Gs;
    Gs  = smithG_GGX_aniso(NdotL, dot(L, X), dot(L, Y), ax, ay);
    Gs *= smithG_GGX_aniso(NdotV, dot(V, X), dot(V, Y), ax, ay);

    // sheen
    vec3 Fsheen = FH * sheen * Csheen;

    // clearcoat (ior = 1.5 -> F0 = 0.04)
    float Dr = GTR1(NdotH, mix(.1,.001,clearcoatGloss));
    float Fr = mix(.04, 1.0, FH);
    float Gr = smithG_GGX(NdotL, .25) * smithG_GGX(NdotV, .25);

    return ((1/PI) * mix(Fd, ss, subsurface)*Cdlin + Fsheen)
        * (1-metallic)
        + Gs*Fs*Ds + .25*clearcoat*Gr*Fr*Dr;
}

::end shader
```
这里将其一步步拆分，详细研究它的实现原理。
## 计算brdf颜色分量（反射率R，F0）
漫反射的颜色，
```
vec3 Cdlin = mon2lin(baseColor);

vec3 mon2lin(vec3 x)
{
    return vec3(pow(x[0], 2.2), pow(x[1], 2.2), pow(x[2], 2.2));
}
```
对baseColor进行gamma校正的结果，因为渲染时颜色空间应为线性空间。

去除漫反射颜色的亮度信息，只保留色相和饱和度，因为后续镜面反射颜色的计算不需要亮度信息。
```
    float Cdlum = .3*Cdlin[0] + .6*Cdlin[1]  + .1*Cdlin[2]; // luminance approx.

    vec3 Ctint = Cdlum > 0 ? Cdlin/Cdlum : vec3(1); // normalize lum. to isolate hue+sat
```

镜面反射颜色，
```
vec3 Cspec0 = mix(specular*.08*mix(vec3(1), Ctint, specularTint), Cdlin, metallic);
```
这里做了两次混合，第一次根据specularTint混合vec3(1)和Ctint，这里用到了specularTint参数，它将对**非金属**的镜面反射颜色进行控制，当它为0时，非金属的镜面反射颜色为白色，这是物理正确的，当它为1时，非金属的镜面反射颜色为Ctint，即baseColor去掉亮度。
上面对specularTint参数已经做了说明，这是一个对美术进行让步的参数，如果严格按照物理正确，非金属的镜面反射颜色应该为白色，而有了这个参数就可以对其进行控制。
第二次混合是根据metallic参数对非金属的镜面反射颜色和Cdlin进行混合，这很好理解，对于金属来说，其镜面反射颜色与漫反射颜色是相同的。
需要注意的是，非金属的镜面反射颜色还有specular*.08这个系数，也就是使用specular参数来控制它的镜面反射强度。specular参数是用来代替折射率IOR的，如果没有这个参数，通常应该使用[[反射模型#菲涅尔方程]]结合IOR来计算反射光线的强度。
0.08这个系数则是一个实验总结的系数，默认情况下IOR为1.5，此时使用菲涅尔方程计算出的反射比例F0应为0.04，这表示当光线沿法线方向入射时，物体会反射4%的能量（将ior=1.5和$\theta_i=\theta_o=0$带入菲涅尔方程可以计算出来）。specular参数的取值为0到1，默认为0.5，此时计算出的Cspec0正好就是0.04。所以系数0.08就是通过specular默认值和IOR默认值反推出来的。
从这里也可以看出，对于非金属，其F0应为0到0.08之间（忽略specularTint参数），对于金属，F0等于漫反射率R。

sheen颜色，它也属于漫反射
```
vec3 Csheen = mix(vec3(1), Ctint, sheenTint);
```
光泽的颜色，通过sheenTint控制，和specularTint是类似的。

这一步计算出来各个brdf的颜色以及反射率，包括漫反射率Cdlin（R），镜面反射率Cspec0（F0）。
## diffuse
如果使用最简单的Lambertian漫反射，diffuse项很好计算，
```
diffuse = Cdlin / Pi;
```
然而disney中的漫反射更复杂一些，它的计算公式为，
$$
f_d=\frac{baseColor}{\pi}(1+(F_{D90}-1)(1-\cos\theta_l)^5)(1+(F_{D90}-1)(1-\cos\theta_v)^5)
$$
其中，
$$
F_{D90}=0.5+2roughness\cos^2\theta_d
$$
代码实现：
```
    float FL = SchlickFresnel(NdotL), FV = SchlickFresnel(NdotV);
    float Fd90 = 0.5 + 2 * LdotH*LdotH * roughness;
    float Fd = mix(1.0, Fd90, FL) * mix(1.0, Fd90, FV);


float SchlickFresnel(float u)
{
    float m = clamp(1-u, 0, 1);
    float m2 = m*m;
    return m2*m2*m; // pow(m,5)
}
```
这里并没有直接乘以$\frac{baseColor}{\pi}$，这项工作放到最后面来完成。

disney漫反射相比于Lambertian漫反射，引入了SchlickFresnel项，这是一个基于观察经验的漫反射模型，它在物体边缘有更亮的漫反射效果，并且引入粗糙度对漫反射的影响。关于它更详细的解释，可以参考[【渲染】Disney BSDF 深度解析](https://zhuanlan.zhihu.com/p/407007915)。

## subsurface
次表面散射被看作是漫反射的一部分，可以理解为光线进入物体后，在物体内部弹射的现象。
这一项是从Hanrahan-Krueger brdf近似而来，模拟次表面散射的效果，其公式为
$$
\begin{aligned}
f_{ss}&=1.25((\frac{1}{\cos\theta_l \cos\theta_v}-0.5)F_{ss}+0.5) \\
F_{ss}&=(1+(F_{SS90}-1)(1-\cos\theta_l)^5)(1+(F_{SS90}-1)(1-\cos\theta_v)^5) \\
F_{SS90}&=rougnhess * \cos^2\theta_l
\end{aligned}
$$
实现代码：
```
    float Fss90 = LdotH*LdotH*roughness;
    float Fss = mix(1.0, Fss90, FL) * mix(1.0, Fss90, FV);
    float ss = 1.25 * (Fss * (1 / (NdotL + NdotV) - .5) + .5);
```
FL和FV在diffuse项已经计算出来了。
## specular
specular项的计算依然使用微平面模型，在[[反射模型#Torrance–Sparrow模型]]中已经做过介绍，但是在实时渲染时，考虑到性能因素，并不能使用功能那么复杂的分布函数。
### Schlick Fresnel
这是用来代替菲涅尔项的近似函数，其形式为
$$
F_{Schlick}=F_0+(1-F_0)(1-\cos\theta_d)^5
$$
这是实时渲染领域使用得最广泛的菲涅尔函数了，其中F0在上面各颜色分量计算时介绍过，表示光线沿法线方向入射时反射能量比例，对于非金属默认是0.04。
这个函数在上面diffuse和subsurface计算时使用过其变体，只是1和F0的位置调换了，简单理解就是随着$\theta_d$从0度变化到90度，$(1-\cos\theta_d)^5$从0变到1，物体反射的能量从F0变化到1，即光线沿法线方向入射时反射能量比例为F0，而光线以法线垂直方向入射时反射能量比例为1，正好对应掠射时发生全反射的物理现象。而diffuse和subsurface获取的能量是反射后剩余的，所以它们的变化趋势与$F_{Schlick}$函数相反，所以1和Fd90（或Fss90）的位置调换了。
disney BRDF使用SchlickFresnel函数加mix函数实现Schlick Fresnel，
```
    float FH = SchlickFresnel(LdotH);
    vec3 Fs = mix(Cspec0, vec3(1), FH);
```
### 法线分布函数 GTR
disney BRDF使用的法线分布函数是GTR分布，其具体形式如下：
$$
D_{GTR}(\theta_h)=\frac{(\gamma - 1)(\alpha^2-1)}{\pi(1-(\alpha^2)^{1-\gamma})}\frac{1}{(1+(\alpha^2-1)\cos^2\theta_h)^\gamma}
$$
其中参数$\gamma$用来控制函数的下降速度，$\gamma$越大，随着$\theta_h$的增大，函数趋近0的速度越快，$\alpha$表示粗糙度。
通常情况下只会使用$\gamma=1$和$\gamma=2$两个函数，前者用来计算clearcoat的specular，后者用来计算物体材质的specular。
可以发现$\gamma=1$时该函数是未定义的，它被替换为如下形式，
$$
D_{GTR_1}(\theta_h)=\frac{\alpha^2-1}{\pi\log\alpha^2}\frac{1}{(1+(\alpha^2-1)\cos^2\theta_h)}
$$
当$\gamma=2$时，它为以下形式，
$$
D_{GTR_2}(\theta_h)=\frac{\alpha^2}{\pi}\frac{1}{(1+(\alpha^2-1)\cos^2\theta_h)^2}
$$
如果考虑各向异性，就使用[[反射模型#微平面的法线分布函数]]中介绍的Trowbridge-Reitz模型，为了使算法更高效，其被替换成如下形式：
$$
D_{GTR_2aniso}(\theta_h)=\frac{1}{\pi}\frac{1}{\alpha_x\alpha_y}\frac{1}{((h\cdot x)^2/\alpha_x^2+(h\cdot y)^2/\alpha_y^2+(h\cdot n)^2)^2}
$$
其中x和y表示该点的切线和副切线，它们与法线n组成一个正交基。
几个函数的实现如下：
```
float GTR1(float NdotH, float a)
{
    if (a >= 1) return 1/PI;
    float a2 = a*a;
    float t = 1 + (a2-1)*NdotH*NdotH;
    return (a2-1) / (PI*log(a2)*t);
}

float GTR2(float NdotH, float a)
{
    float a2 = a*a;
    float t = 1 + (a2-1)*NdotH*NdotH;
    return a2 / (PI * t*t);
}

float GTR2_aniso(float NdotH, float HdotX, float HdotY, float ax, float ay)
{
    return 1 / (PI * ax*ay * sqr( sqr(HdotX/ax) + sqr(HdotY/ay) + NdotH*NdotH ));
}
```
### 几何函数 Smith-GGX
Smith-GGX随粗糙度的变化更加平滑，其形式为
$$
\begin{aligned}
G(l,v,h)=G_{GGX}(l)G_{GGX}(v) \\
G_{GGX}(v)=\frac{2(n \cdot v)}{(n \cdot v)+\sqrt{\alpha^2+(1-\alpha^2)(n \cdot v)^2}}\\
\alpha = (0.5+roughness/2)^2
\end{aligned}
$$
这个函数有一个特别的地方，它的分子是$2(n \cdot v)$，如果再将$G_{GGX}(l)$考虑进去，它的分子就变成了$4(n \cdot v)(n \cdot l)$，正好与Torrance–Sparrow模型的分母相同，所以此项可以直接约分。这也是使用Smith-GGX的优点之一，最终的光照计算不需要除以$4(n \cdot v)(n \cdot l)$，减少了计算量。
代码实现：
```
float smithG_GGX(float NdotV, float alphaG)
{
    float a = alphaG*alphaG;
    float b = NdotV*NdotV;
    return 1 / (NdotV + sqrt(a + b - a*b));
}
```
注意这里已经去掉分子了。
同时，它有一个各项异性的版本，
$$
G(v) = \frac{2(n \cdot v)}{n·v + \sqrt{(x·v*\alpha_x)^2 + (y·v * \alpha_y)^2 + (n·v)^2}}
$$
和法线分布函数一样，x和y表示其切线和副切线，对应实现如下：
```
float smithG_GGX_aniso(float NdotV, float VdotX, float VdotY, float ax, float ay)
{
    return 1 / (NdotV + sqrt( sqr(VdotX*ax) + sqr(VdotY*ay) + sqr(NdotV) ));
}
```
同样去掉分子。
### specular的计算
有了上面三个函数，就可以计算brdf的specular部分了，
```
    // specular
    float aspect = sqrt(1-anisotropic*.9);
    float ax = max(.001, sqr(roughness)/aspect);
    float ay = max(.001, sqr(roughness)*aspect);
    float Ds = GTR2_aniso(NdotH, dot(H, X), dot(H, Y), ax, ay);
    float FH = SchlickFresnel(LdotH);
    vec3 Fs = mix(Cspec0, vec3(1), FH);
    float Gs;
    Gs  = smithG_GGX_aniso(NdotL, dot(L, X), dot(L, Y), ax, ay);
    Gs *= smithG_GGX_aniso(NdotV, dot(V, X), dot(V, Y), ax, ay);
```
首先通过anisotropic和roughness计算出不同方向的粗糙度ax，ay，然后使用它们计算其他3个函数。
Ds使用GTR2_aniso即GTR法线分布函数的各项异性版本计算。
Fs使用SchlickFresnel和mix插值计算。
Gs分为两个方向，分布计算光源方向和观察方向的遮蔽效果，都使用使用smithG_GGX_aniso函数。
## sheen
光泽通常用来模拟天鹅绒、布料这类材质外表的光泽效果，它们通常拥有较大的掠射效应，材质边缘呈现出更强的光线反射效果。
其计算方式很简单：
```
vec3 Fsheen = FH * sheen * Csheen;
```
FH是SchlickFresnel项，即$(1-\cos\theta_l)^5$，然后乘以光泽强度和光泽颜色。
## clearcoat
clearcoat可以视为物体材质上的第二个高光反射，具体的计算方法与specular相似，
```
    // clearcoat (ior = 1.5 -> F0 = 0.04)
    float Dr = GTR1(NdotH, mix(.1,.001,clearcoatGloss));
    float Fr = mix(.04, 1.0, FH);
    float Gr = smithG_GGX(NdotL, .25) * smithG_GGX(NdotV, .25);
```
clearcoat同样计算D、F、G三项，但是其中的有的参数被固定了。
计算Dr时使用各项同性的GTR1分布，但是使用clearcoatGloss代替了roughess，具体来说，
roughess = 1 - clearcoatGloss，并且这里保证其最小值不会小于0.001。
计算Fr时，F0被固定为0.04，对应常用的IOR = 1.5。
计算Gr时，使用各项同性的smithG_GGX，并且其粗糙度被固定为0.25。
## 最终BRDF计算
将上面所有的项组合为最终结果，
```
    return ((1/PI) * mix(Fd, ss, subsurface)*Cdlin + Fsheen)
        * (1-metallic)
        + Gs*Fs*Ds + .25*clearcoat*Gr*Fr*Dr;
}
```
在这里可以看到，漫反射项Fd和次表面散射项ss通过subsurface参数混合，这意味着它们的能量总和是固定的，最终的结果需要乘以Cdlin / PI（漫反射率）。
Fsheen作为漫反射的一部分加入漫反射项的计算结果。
漫反射项会乘以1 - metallic，这表示当金属度为1时，漫反射会全部消失，对应金属材质没有漫反射的物理特性。
后面的specular和clearcoat直接加上即可，对于clearcoat，这里加上了系数clearcoat来控制其效果强度，前面的0.25应该是固定的美术调节参数，防止clearcoat效果过强，因为clearcoat这一项是没有考虑能量守恒的。
# disney BSDF
在disney BRDF后，disney又针对其做了扩充，提出了disney BSDF，相比与原来的BRDF，BSDF增加了透射部分，并且对于次表面散射部分有了更加完善的实现。
上面提到过，BRDF的反射模型主要分为两部分，金属与非金属，
![[Pasted image 20250818182505.png]]
而BSDF引入了新的透射BSDF，
![[Pasted image 20250818182903.png]]
原本的dielectric BRDF需要和specular BSDF混合之后，才和metallic BRDF混合成最终BSDFF。
## specular BSDF
作为新增项，specular BSDF引入了两个新参数：
- IOR：折射率，控制光线反射的能量比例，在BRDF中它由specular替代，现在需要计算透射部分，又将其使用回来。
- specTrans（Transmission）：控制透射光线能量比例，即透射效果强弱。
在BSDF中，严格来说光线的能量被分为3部分，
$$
\begin{align*}
\text{反射能量} &: \quad E_{\text{refl}} = F \\
\text{透射能量} &: \quad E_{\text{trans}} = (1-F) \cdot \text{specTrans} \\
\text{散射能量} &: \quad E_{\text{sss}} = (1-F) \cdot (1-\text{specTrans}) 
\end{align*}
$$
反射的能量F由菲涅尔定律计算出。
透射的能量为反射剩余的能量乘以specTrans系数。
最终剩下散射的能量，包括次表面散射部分和漫反射部分。
按照这个思路，specular BSDF将与diffuse + subsurface部分根据specTrans参数进行融合。
关于specular BSDF的计算方法，在[[常用的反射模型#粗糙的绝缘体]]中有推导，同样使用了微平面的理论。
specular BSD通常F只会在离线渲染中使用，第一，其计算方法太复杂，第二，只有在光线追踪算法中，才可以模拟光线透过物体时的渲染效果，在实时渲染中通常只使用alpha来实现半透明效果，要想实现更丰富的透射效果（如光线折射，磨砂玻璃），需要使用其他的技术手段模拟。
## subsurface
之前BRDF使用的subsurface模型过于简单，disney在BSDF的扩充中对subsurface的计算进行了更加详细的分析。
首先，在之前的模型中，漫反射被分成了两个部分，diffuse和subsurface，这是一个理想光滑表面的反射方式，
![[Pasted image 20250826185254.png]]
在粗糙的微平面下，漫反射应该是这样的，
![[Pasted image 20250826185711.png]]

光线照射到粗糙物体之上后，在表面先形成retro-reflection和sheen两种反射，然后进入物体内部，在其中不断弹射形成subsurface，弹射出的光线形成diffuse。
这样就可以用更加精细的模型描述漫反射现象了，在BSDF中，diffuse项做了以下修改，
$$
\begin{aligned}
f_d=\frac{R}{\pi}(1-0.5F_L)(1-0.5F_V)+f_{retro-reflection} \\
f_{retro-reflection}=\frac{R}{\pi}R_R(F_L+F_V+F_LF_V(R_R-1))
\end{aligned}
$$
其中
$$
F_L=(1-\cos\theta_l)^5,\space F_V=(1-\cos\theta_v)^5, \space R_R=2*roughness*\cos^2\theta_d
$$
增加了retro-reflection的部分，sheen部分依然使用之前的计算方式。
次表面散射部分不再使用Hanrahan-Krueger brdf的近似，而是使用物理正确的BSSRDF近似，
与BRDF不同，BSSRDF公式更加复杂，
$$
S(x_i,\omega_i,x_o,\omega_o)=CF_t(x_i,\omega_i)R(|x_o-x_i|)F_t(x_o,\omega_o)
$$
与$f_r(p,\omega_i,\omega_o)$相比，BSSRDF多了一个参数，这是因为散射时入射点和出射点的位置不一样。
C为漫反射率R（baseColor）。
Ft表示光线入射和出射时的折射系数，表示折射的光线能量大小。
R被称为扩散曲线（Diffusion Profile），它表示一束光垂直照射到材质表面一个无限小的点上，在距离该点**径向距离为 `r`（$|x_o-x_i|$）** 的另一个点处，光出射的**相对概率分布**。简单来理解光从入射点进入后，最有可能在多远的地方穿出来。所以当计算一个点的光照结果时，可以使用这个函数统计出周围哪些点可能对这个点也有光照贡献（光线从那些点进入，在当前点射出）。
disney bsdf使用的Diffusion Profile公式为
$$
R_d(r)=\frac{e^{-\frac{r}{d}}+e^{-\frac{r}{3d}}}{8\pi dr}
$$
r表示扩散距离，d表示散射距离，通常作为一个参数暴露给美术，它是一个float3类型的值，因为不同的色光在物体中的散射距离是不一样的。
这个公式的优点是在无限大平面的积分为1，并且pdf和cdf非常容易求出，所以很方便进行采样，
$$
\int_0^{2\pi}\int_0^{\infty}R_d(r)rdrd\phi=1
$$
如果计算出了$R_d$，结合Ft就可以计算最终的BSSRDF值。
关于$R_d$的计算，由于它的含义为光线出射点相对于入射点的相对概率分布，计算时可以根据它的分布在物体周围对入射点进行采样（通过重要性采样的方法），再采样出其入射方向（和通常的入射方向采样相同，比如可以使用半球面的$\cos\theta$采样）。出射点和入射点之间的距离带入公式即可计算出$R_d$.
同时，有了入射和出射方向的信息，也可以结合折射率计算出$F_t(i)$和$F_t(o)$，这样BSSRDF就可以计算出来了。
与BRDF相比，BSSRDF多了一次对入射点的采样。
在实时渲染中，通常使用屏幕空间的方法实现次表面散射效果（这种技术叫做屏幕空间次表面散射，简称sssss），同样根据$R_d$函数，在一个屏幕上圆盘之中对入射点进行采样，然后根据距离计算出$R_d$，后面就和光线追踪算法一样了。由于使用的是屏幕空间的采样，精确度相比光线追踪要差一些。
次表面散射引入了一个新的参数：
- scatterDistance（散射距离）:float3类型，指光在物体内部平均传输距离，越大次表面散射的效果就越平滑。大多数实现会将此参数拆分成两个参数，一个Color和一个Distance，二者相乘获得scatterDistance。
## 关于IOR
bsdf引入的新参数IOR(折射率)对美术来说并不友好，之前是使用specular参数来控制。关于specular与IOR的关系在上面[[#计算brdf颜色分量（反射率R，F0）]]，默认的specular = 0.5对应IOR = 1.5 对应 F0 = 0.04，specular取值范围为0到1，对应IOR范围为1到1.8。
由于bsdf引入了透射部分，必须保证计算反射和透射时能量时IOR相同的，所以IOR参数不得不使用回来。不过IOR参数和F0近似于线性关系，所以它依然可以做线性的混合。对于大多数材质IOR取值在1到2之间，默认为0.05。
## Fresnel函数
在brdf中使用Schlick Fresnel作为Fresnel的近似函数，它计算简单，效果也很好。不过如果考虑到透射部分，其Fresnel项需要更精确的结果，因为F直接决定了反射光、透射光、散射光的能量大小，对bsdf引入的specular Transmission和subsurface都很重要。
有了IOR之后，可以通过下面方式来计算透射的Fresnel项。
$$
\begin{aligned}
F_0&=(\frac{1-\eta}{1+\eta})^2 \\
\cos^2\theta_t&=1-\frac{1-\cos^2\theta_i}{\eta^2} \\
F_{Schlick}(\theta_i,\eta)&=\begin{cases} F0+(1-F0)(1-\cos\theta_t)^5 &&  \cos^2\theta_t > 0 \\ 1 && otherwise\end{cases}
\end{aligned}
$$
当然，有了IOR和折射光的方向之后，最好的方法就是直接使用[[反射模型#菲涅尔方程]]来计算Fresnel项，在[[常用的反射模型]]中就是这样做的。为了提高性能，除了透射之外的其他项依然可以使用Schlick Fresnel近似。
## 关于薄面
对于薄面的BSDF，disney新引入了diffTrans，它用来插值diffuse reflection和diffuse transmission项，diffuse reflection依然使用漫反射的计算方式，diffuse transmission可以使用简单的Lambertian来计算。
同时，薄面的次表面散射效果很弱，依然使用之前brdf的近似来实现，但是为了与subsurface进行区分，使用flatness来插值漫反射项和subsurface项。
同时，薄面的透射引起的光线方向偏移几乎可以忽略不记，但是会增强模糊效果，所以通过折射率来对粗糙度进行校正，具体方法是$roughness = (0.5 * \eta - 0.35) * roughness$。
# Reference
https://zhuanlan.zhihu.com/p/60977923
https://blog.selfshadow.com/publications/s2015-shading-course/burley/s2015_pbs_disney_bsdf_notes.pdf

