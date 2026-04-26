## 简介
基于Renderdoc截帧学习少女前线2角色卡通渲染，拆解材质系统、光照模型，并尝试在Unity中进行还原。

## 最终效果
全身展示

上半身细节展示

## 方案总结
GF2 的角色渲染核心思路是 **PBR + NPR 结合**。在保留物理正确的光照基础上，通过 Ramp 贴图、SDF 阴影等手段对视觉表现进行卡通化改造，兼顾质感真实与二次元风格。
其中，衣服、丝袜等材质使用了结合Ramp的PBR光照算法，而脸、头发、眼睛等材质则使用了 NPR 的方式来渲染，充分还原二次元卡通渲染的特征。

整体光照计算管线可以概括为以下四个阶段：

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
衣服（含枪械）是所有材质中最接近标准 PBR 的，但在每个光照阶段都通过 Ramp 贴图进行了卡通化改造。以Cloth02为例，使用贴图如下：
| 基础色贴图 | 法线（DXT5nm） | RMO | Ramp |
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
- 先计算标准 GGX BRDF 三要素：**NDF（法线分布函数）、Smith G（几何遮蔽函数）、Fresnel（菲涅尔）**
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
- 再计算 **NDF_thin、Smith G_thin**，取 **saturate((NDF * G) / (NDF_thin * G_thin)) 的比值结果作为 Ramp 贴图的 U 坐标采样**
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
### 脸
### 头发
### 眼睛（眼球、眼部高光、眼部压暗）
### 前发投影/前发半透
### 描边
### LUT

## 后续迭代
### Bloom
### TAA
### 角色高精度阴影
### GTAO 屏幕环境光遮蔽
