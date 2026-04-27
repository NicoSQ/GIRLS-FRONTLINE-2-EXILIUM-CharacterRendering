## 简介
记录一下基于 Renderdoc 截帧学习少女前线2角色卡通渲染，拆解材质系统和光照模型，并在 Unity 中进行效果还原。  
GF2 原生使用**延迟管线**渲染，本次还原基于快速复刻角色渲染效果，使用**前向管线**还原。

## 最终效果
全身展示

上半身细节展示

## 方案总结
GF2 的角色渲染核心思路是 **PBR + NPR 结合**。部分材质在保留物理正确的光照基础上，通过 Ramp 贴图、SDF 阴影等手段对视觉表现进行卡通化改造，兼顾质感真实与二次元风格。
其中，衣服、丝袜等材质使用了结合 Ramp 的 PBR 光照算法，而脸、头发、眼睛等材质则使用了 NPR 的方式来渲染，充分还原二次元卡通渲染的特征。

整体光照计算管线可以大致概括为以下四个阶段：

| 阶段 | PBR 材质（衣服/丝袜） | NPR 材质（脸/头发） |
|------|---------------------|---------------------|
| 直接光漫反射 | NdotL → Ramp 采样 | SDF 阴影 / NdotL → Ramp |
| 环境漫反射 | SampleSH + NormalizeSH | 同左 |
| 直接光高光 | GGX BRDF → Ramp 调制 | SDF 通道 / 高光贴图 UV 偏移 |
| 环境高光 IBL | Ramp 菲涅尔（仅背光侧） | 同左 |

## 贴图分析
角色使用的贴图类型及其用途：

| 贴图类型 | 使用材质 | 说明 |
|---------- |---------|------|
| 基础色贴图 | 衣服、丝袜、脸、头发、眼球 | 物体固有色 |
| 法线贴图 | 衣服、丝袜 | 表面细节凹凸，DXT5nm 格式压缩（法线 XY 存储在 AG 通道） |
| RMO 贴图 | 衣服、丝袜 | R=Roughness 粗糙度、G=Metallic 金属度、B=Occlusion 环境光遮蔽 |
| Ramp 贴图 | 衣服、丝袜、脸、头发 | 不同行控制不同光照阶段的色阶过渡。第1行 → 直接光漫反射，第2行 → 直接光高光，第3行 → 环境高光IBL，第4行 → 额外光源直接光漫反射 |
| SDF 贴图 | 脸 | R=主阴影阈值、G、B=鼻尖、唇角高光 |
| 高光贴图 | 头发 | 美术绘制的头发高光分布和强度 |

其中 Ramp 贴图是整个渲染管线中最核心的贴图资产。美术通过编辑 Ramp 贴图的不同行，即可直接控制漫反射色阶、高光颜色过渡、菲涅尔边光等视觉效果，无需修改 Shader 代码。

## 材质解析
### 衣服
衣服（含枪械）是所有材质中最接近标准 PBR 的，但在每个光照阶段都使用 Ramp 贴图进行了处理。以Cloth02为例，使用贴图如下：
| 基础色贴图 | 法线贴图（DXT5nm） | RMO贴图 | Ramp贴图 |
|------|------|------|------|
| <img width="256" height="256" alt="c_CheetaSR01_slg_cloth02_d" src="https://github.com/user-attachments/assets/a702ff42-fb85-497d-90fe-28bfc6873f8d" /> | <img width="256" height="256" alt="c_CheetaSR01_slg_cloth02_n" src="https://github.com/user-attachments/assets/7a163655-b2ed-4316-bb18-62ee63fcb686" /> |<img width="256" height="256" alt="c_CheetaSR01_slg_cloth02_rmo" src="https://github.com/user-attachments/assets/4028a5fa-e9ab-4d09-bea2-fc060ca01c28" /> | <img width="256" height="16" alt="RampMap_Linear_RGBAHalf (2)" src="https://github.com/user-attachments/assets/ecd8dc82-f3c0-495e-9f01-ed630b620184" /> |

0. **F0计算**：计算PBR基础的F0参数
```hlsl
// diffColor 相比于 Unity 少乘 0.96
float3 diffColor = albedo * (1.0 - metallic);
float3 F0 = lerp(0.04, albedo, metallic);
```
1. **直接光漫反射**：计算 NdotL * shadow 的结果，用结果作为 U 坐标采样 Ramp 贴图第一行。
```hlsl
float3 directDiffuse = mainShadow * NdotL;
// 将 directDiffuse 钳制最小值 6.1e-5
float diffLumClamped = max(directDiffuse.z, 6.1e-5);
// Ramp 漫反射 (V=0.125)
float2 rampDiffUV = float2(diffLumClamped, 0.125);
float4 rampDiff = SAMPLE_TEXTURE2D(_RampMap, sampler_RampMap, rampDiffUV);
directDiffuse = rampDiff.rgb;
```
**（补充图片）**  

2. **直接光高光（GGX BRDF + Ramp 调制）**：  
- 先计算标准 GGX BRDF 三要素：**NDF（法线分布函数）、Smith G（几何遮蔽函数）、Fresnel（菲涅尔）**。
```hlsl
// GGX BRDF 计算
// ── GGX NDF (D) ──
float roughSq = roughness * roughness;            
float oneMinusNdotHSq = 1.0 - NdotH * NdotH;     
float dTerm = roughSq * NdotH;                    
float dDenom = dTerm * dTerm + oneMinusNdotHSq;    
float D = roughSq / dDenom;                                                                  
D = min(D * D, 2048.0);

// ── Smith GGX Geometry (G) ──
// 使用不带 sqrt 的近似，性能更优
float oneMinusRoughSq = 1.0 - roughness * roughness; 
float gTermV = NdotV * oneMinusRoughSq + roughSq; 
float gTermL = NdotL * oneMinusRoughSq + roughSq;     
float gDenom = gTermL * NdotV;                     
gDenom = NdotL * gTermV + gDenom;                      
gDenom = max(gDenom, 0.0001);                          
float G = 1.0 / gDenom;                                                                            
G = min(G * 0.5, 1.0);

// ── Schlick Fresnel (F) ──
float oneMinusVdotH = max(1.0 - VdotH, 0.001);                                 
float pow2 = oneMinusVdotH * oneMinusVdotH;            
float pow4 = pow2 * pow2;                               
float pow5 = oneMinusVdotH * pow4;                           
float F_schlick = (saturate(F0.y * 50.0) - 1) * pow5 + 1.0;       
```
- 再计算 **NDF_thin、Smith G_thin**，取 **saturate((NDF * G) / (NDF_thin * G_thin)) 的比值结果作为 Ramp 贴图的 U 坐标采样**。
```hlsl
float LdotH = saturate(dot(mainLightDirection, halfDir));

// D_thin (roughness⁴ 版)，更锐利的NDF，效果仅取决于粗糙度
float roughSq4 = roughSq * roughSq;
float D_thin = roughSq / roughSq4;
D_thin = min(D_thin * D_thin, 2048.0);

// G_thin (使用 VdotH & LdotH 代替 NdotV & NdotL)
float gThinV = VdotH * oneMinusRoughSq + roughSq;
float gThinL = LdotH * oneMinusRoughSq + roughSq;
float gThinDenom = VdotH * gThinL;
gThinDenom = LdotH * gThinV + gThinDenom;
gThinDenom = max(gThinDenom, 0.0001);
float G_thin = 1.0 / gThinDenom;
G_thin = min(G_thin * 0.5, 1.0);

// rampSpecU = D * G / (D_thin * G_thin)
float rampSpecU = saturate((D * G) / (D_thin * G_thin));

// Ramp 高光采样 (V=0.375)
float2 rampSpecUV = float2(rampSpecU, 0.375);
float4 rampSpec = SAMPLE_TEXTURE2D(_RampMap, sampler_RampMap, rampSpecUV);
specular = D_thin * G_thin * F_schlick * rampSpec.rgb;

specular = max(specular, 0.0);                            
specular = min(specular, 10.0);
// 输出直接光高光                        
specular = F0 * specular;
```
- **效果**：保留了 GGX 物理正确的高光形状，结合 Ramp 将物理高光"翻译"成美术可控的色阶过渡。

3. **环境漫反射**：使用 Unity SampleSH 采样球谐光照，加入 NormalizeSH 归一化处理，**把SH环境光的方向性去掉，只保留整体的平均亮度和色调**。
```hlsl

```
4. **环境高光 IBL（Ramp 菲涅尔）**：
- 核心创新：用物体空间的方向信息判断暗面位置，让菲涅尔边光只出现在背光侧
- Ramp 贴图控制边光的形态和过渡
- 与 Unity 原生 IBL 的区别：Unity 菲涅尔全方向均匀，GF2 有选择性地增强背光侧的边光
### 丝袜  
**丝袜材质**使用的贴图和**衣服材质**完全相同。同时在**衣服材质**的 PBR 计算基础上添加了**基于视角变化的直接光漫反射 + 各项异性高光**，其余阶段（环境漫反射、直接光漫反射、环境高光 IBL）与衣服材质完全一致。
1. **基于视角变化的直接光漫反射**：漫反射颜色会随视角变化。在 F0 计算阶段引入视角依赖的颜色偏移。
2. **各项异性高光（替代传统 GGX 各向同性高光）**：使用各向异性变体的 NDF，**产生沿法线方向拉伸的线状高光，视觉上呈现丝袜特有的细长高光条纹**。其中，G 项和 F 项仍与标准 GGX 相同。
```hlsl

```
**（补充图片）**  

### 脸  
GF2 脸部渲染采用 NPR 的 SDF 贴图方案。使用贴图如下：
| 基础色贴图 | SDF贴图(R->阴影 G B->鼻尖、唇边高光) | Ramp贴图 |
|------|------|------|
| <img width="256" height="256" alt="c_CheetaSR01_slg_face_d" src="https://github.com/user-attachments/assets/38a46ee5-536c-4026-9f6b-96a567b2c180" /> | <img width="256" height="256" alt="A_face_b" src="https://github.com/user-attachments/assets/4d3f98ec-7e79-4410-9d23-908d10170c86" /> | <img width="256" height="16" alt="RampMap_Linear_RGBAHalf" src="https://github.com/user-attachments/assets/78df5018-388b-48a6-9306-91c3b37d9a90" /> |  
1. **脸部 SDF 阴影**：由 SDF 贴图的 R 通道存储的面部阴影阈值控制，配合 Ramp 贴图实现亮暗面可控的面部明暗过渡。
2. **脸部高光**：SDF 贴图的 G、B 通道控制鼻尖和唇角高光。
3. **环境光（漫反射、高光）**：跟衣服计算基本一致。
```hlsl

```

**（补充图片）**  

### 头发
GF2 头发采用 NPR 渲染，分为**前发**和**后发**两个部分，在光照计算上二者相同。  
其中，头发的**直接光漫反射，环境光（漫反射、高光）部分与衣服、丝袜计算基本一致，主要差异在直接光高光部分**。
| 基础色贴图 | 高光贴图(_MatCapMap) | Ramp贴图 |
|------|------|------|
|<img width="256" height="256" alt="c_CheetaSR01_slg_hair_d" src="https://github.com/user-attachments/assets/72a2c68e-39f5-42d8-9741-211927b89afa" /> |<img width="256" height="256" alt="c_CheetaSR01_slg_hair_spc" src="https://github.com/user-attachments/assets/5e743dee-0552-44f7-9795-32b01cebda88" /> | <img width="256" height="16" alt="RampMap_Linear_RGBAHalf (4)" src="https://github.com/user-attachments/assets/226eaa9f-4dac-452c-b0df-c867c509f070" /> |   
  
头发模型制作了两套 UV，用 UV0 采样基础色贴图，UV1 采样高光贴图。  
1. **头发高光**：高光贴图的RGB通道存储前后发的高光分布，A通道存储高光遮罩。**基于视角方向对采样 UV 的 Y 向量进行偏移，模拟头发反光随视角的动态变化**。
```hlsl
// 基于视角方向对采样 UV 的 Y 向量进行偏移，模拟头发各向异性高光随视角上下滑动
float2 matcapUV;
matcapUV.x = input.uv01.z;
matcapUV.y = input.uv01.w - viewDirWS.y * _MatCapUVOffset;
float NdotV = saturate(dot(viewDirWS, normalWS));

float4 matcapSamp = SAMPLE_TEXTURE2D(_MatCapMap, sampler_MatCapMap, matcapUV);
float3 matcapSpec = NdotV * matcapSamp.rgb * _MatCapIntensity;
// 完全阴影区域保留 10% 的高光亮度，避免高光完全消失         
float3 matcapContrib = matcapSpec * (0.1 + 0.9 * (NdotL * shadow)) / PI;
```
**（补充图片）**

### 眼睛（眼球、眼部高光、眼部压暗）  
GF2 的眼睛由**三个独立材质层**叠加实现，可以分开独立调节： 
| 眼球贴图 | 眼部压暗/高光贴图 |
|------|------|
| <img width="256" height="256" alt="c_CheetaSR01_slg_eye_d" src="https://github.com/user-attachments/assets/2fc13609-1c15-4686-8ac1-203bf26c307d" /> | <img width="256" height="256" alt="c_CheetaSR01_slg_eyeblend" src="https://github.com/user-attachments/assets/1d793d93-95a4-431e-b754-ba30ea24a843" /> |
1. **眼球材质**：根据 View 方向做轻微 UV 偏移，模拟视差效果，增加眼球的立体感和灵动感。
2. **眼部压暗**：Blend 模式`DstColor Zero`，在眼球上方叠加暗色增加深度感
3. **眼部高光**：Blend 模式`One One`，叠加明亮的高光点

**（补充图片）**

### 前发投影/前发半透
通过多 Pass 配合写入不同 Stencil 值实现**前发在脸上的投影和半透效果**。  
1. **脸部基础渲染Pass**：写入基础 Stencil 值（200）
2. **眉毛/眼球渲染Pass**：写入不同的 Stencil 值（204），标记为不可被前发覆盖的区域
3. **前发投影Pass**：
   - **只更新 Stencil，不写入任何颜色**
   - 在顶点着色器中基于光照方向偏移前发顶点位置
   - 偏移后与面部重叠的区域 Stencil + 1（201）
```hlsl
Cull Back
ZWrite Off          // 不写深度
ColorMask 0         // 不写任何颜色 ← 全部 RT 都不写

Stencil
{
    Ref 200                   // 面部的 Stencil 值 (200)
    CompFront Equal           // 只在面部区域（Stencil=200）生效
    PassFront IncrSat         // 面部+头发重叠 → 200→201
    FailFront Keep
    ZFailFront Keep
}

......

Varyings vert(Attributes v)
{
    Varyings o;

    float3 posWS = TransformObjectToWorld(v.positionOS.xyz);

    // 基于光照方向偏移（模拟头发阴影投射）
    Light mainLight = GetMainLight();
    posWS.x += mainLight.direction.x * _HairShadowOffsetX;
    posWS.y += _HairShadowOffsetY;

    o.positionCS = TransformWorldToHClip(posWS);
    return o;
}

```
4. **脸部绘制前发投影半透Pass**：读取上一步更新的 Stencil 值，在标记区域半透叠加阴影色（真正绘制前发阴影的 Pass）
```hlsl
// 前发投影与脸部Blend，可以控制阴影强度
Blend SrcAlpha OneMinusSrcAlpha

Stencil
{
    Ref 201
    CompFront Equal
    PassFront DecrSat   // 通过后：Buffer 值减 1（201 → 200）
    FailFront Keep
    ZFailFront Keep
}

......

// 后续Pass计算跟脸部基础SDF光照绘制一致，输出半透
```
5. **前发半透 pass**：只绘制前发与眉毛/眼球重叠的区域，实现前发遮挡时的半透明效果
```hlsl
// 半透明发梢
Blend SrcAlpha OneMinusSrcAlpha

Stencil
{
    Ref 204      
    CompFront Equal      // 只在前发和眼球、眉毛重叠（Stencil=200）的地方生效
    PassFront Keep
    FailFront Keep
    ZFailFront Keep
}

// 后续Pass计算跟前发基础光照绘制一致，输出半透
```
**（补充图片）**
### 描边  
描边使用基于法线外扩的经典描边方案，配合一些计算实现效果优化。  
- **平滑法线**：顶点色 RGB 通道存储预计算的切线空间下平滑法线，解决硬边处法线突变导致的描边截断问题，A 通道存储描边遮罩，控制描边区域和粗细。
- **外扩流程**：法线转换到裁剪空间下实现描边外扩，实现屏幕宽高比矫正（避免描边在宽屏上被拉伸）和透视补偿（预乘 clip.w 抵消透视除法导致的近粗远细）。
```hlsl
// 模型顶点色RGB通道预存切线空间下平滑法线
float3 vertColor = saturate(input.vertexColor.rgb) * 2.0 - 1.0;
float3 smoothNormalOS = vertColor.x * tangentOS + vertColor.y * bitangentOS + vertColor.z * normalOS;
smoothNormalOS = normalize(smoothNormalOS);
 
float3 smoothNormalWS = TransformObjectToWorldNormal(smoothNormalOS);
float3 csNormal = TransformWorldToHClipDir(smoothNormalWS);
 
float aspect = _ScreenParams.x / _ScreenParams.y;
csNormal.x /= aspect;
csNormal = normalize(csNormal);
 
float outlineWidth = _OutlineWidth * input.vertexColor.w * 0.001302;
// 预乘 clip.w 抵消透视除法导致的近粗远细
// 实际效果仍有近小远大的视觉感受，原因是相机拉远后角色占屏像素变小而描边像素不变，对比之下感觉描边变粗，补救方案：限制 w 的范围（clamp）
float clampedW = clamp(output.positionCS.w, 1.0, _MaxOutlineDistance);
output.positionCS.xy += csNormal.xy * outlineWidth * clampedW;

// 使用世界空间顶点坐标点乘光照XZ方向，计算描边的亮暗面变化
Light mainLight = GetMainLight();
float2 lightDirXZ = normalize(mainLight.direction.xz);
float2 posXZ = normalize(positionWS.xz);
float dirDot = dot(posXZ, lightDirXZ);
float blendFactor = dirDot * 0.5 + 0.5;
float4 outlineColor = lerp(_OutlineShadowColor, _OutlineColor, blendFactor);
```
**（补充图片）**
### LUT
GF2 使用一张自定义的 LUT 图完成最终的色彩映射（**LUT 图结合了 ColorGrading + Tonemapping**）。  
| LUT 图 |
|--------|
|<img width="1024" height="32" alt="image" src="https://github.com/user-attachments/assets/b590c57f-9c31-4d41-89a9-074a54cfb41b" /> |  

使用自定义的 RenderFeature 和 RenderPass，传入这张 LUT 对最终画面进行处理。  
**（补充图片）**

## 后续计划迭代  
### Bloom  
GF2 的后效 Bloom 使用降采样再升采样的方案实现。
首先基于亮度阈值提取屏幕需要发光的像素，然后再对需要发光像素的画面进行逐级分辨率缩小（每级 1/2），扩大发光扩散范围，然后再逐级放大回原始分辨率，根据权重混合各级模糊效果。最后用升采样回来的低频Bloom和直接降采样1/2得到的高频Bloom结果相加，叠加到原始屏幕上。  
**（补充图片）**

### TAA 抗锯齿
GF2 使用 TAA 抗锯齿，完美适配项目原生使用的延迟渲染管线。截帧中，每个光照物体在输出 GBuffer 时都附带输出了 Motion Vector 信息，供后续的 TAA 后处理计算。  
**（补充图片）**

### 角色高精度阴影  
GF2 为角色单独渲染一张高分辨率 ShadowMap。相比全场景 ShadowMap，角色获得更高的阴影精度而不浪费分辨率在场景大面积区域上。  
**（补充图片）**

### GTAO 屏幕环境光遮蔽  
GF2 的角色展示界面计算了基于屏幕的环境光遮蔽，给角色增强了更多细节和立体感。  
**（补充图片）**
