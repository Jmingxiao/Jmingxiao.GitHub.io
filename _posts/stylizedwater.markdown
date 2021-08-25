---
layout: post
title:  "Stylized Water"
date:   2021-08-25 17:00:00 +0530
categories: Unity Shader
---

## Introduction
This is a custom stylized water I created month ago. It contains technic of depthdifference reflections refractions normals, and so on.  
  
Before I create this water shader, I have tried to work on different water materials from unity assets store and other resources. The most convincing one is from [Catlike Coding][catlike-coding].  
With all those reference I've found. I separate the full water shader into different parts.
***
## Water Reflection  
The reflection is basically another perspective camera under the water and the position is xmain,ywater-(Ymain - ywater),zmain; and the eular rotation is -x,y,z.  
This code took the reference of the unity demo Boat Attack.  
plain camera's reflection matrix:
``` csharp
private static void CalculateReflectionMatrix(ref Matrix4x4reflectionMat, Vector4 plane)
{
    reflectionMat.m00 = (1F - 2F * plane[0] * plane[0]);
    reflectionMat.m01 = (-2F * plane[0] * plane[1]);
    reflectionMat.m02 = (-2F * plane[0] * plane[2]);
    reflectionMat.m03 = (-2F * plane[3] * plane[0])
    reflectionMat.m10 = (-2F * plane[1] * plane[0]);
    reflectionMat.m11 = (1F - 2F * plane[1] * plane[1]);
    reflectionMat.m12 = (-2F * plane[1] * plane[2]);
    reflectionMat.m13 = (-2F * plane[3] * plane[1])
    reflectionMat.m20 = (-2F * plane[2] * plane[0]);
    reflectionMat.m21 = (-2F * plane[2] * plane[1]);
    reflectionMat.m22 = (1F - 2F * plane[2] * plane[2]);
    reflectionMat.m23 = (-2F * plane[3] * plane[2])
    reflectionMat.m30 = 0F;
    reflectionMat.m31 = 0F;
    reflectionMat.m32 = 0F;
    reflectionMat.m33 = 1F;
}
```
Or you could also try to use the Screen space reflection technic.  
***
## Water Color 
For the water Color I use a little trick to simulate the water color from shallow to depth.   
```c
//simply create a texture 2D variable called _CameraDepthTexture.

//take the color under water.
fixed4 rt = tex2D(_WaterBackground, normScreenPos.xy);

float depth = tex2D(_CameraDepthTexture, normScreenPos);
depth = LinearEyeDepth(depth);
//depth fade
float depthFade = saturate((depth - IN.screenPos.a)/ _WaterDepth);
rt.a = _ShallowColor.a;
float4 col = lerp(rt, _ShallowColor, rt.a);
col = lerp(col, _DepthColor, depthFade);
```
Foam calculation is fairly easy 
``` c
float foamDiff = saturate((depth - IN.screenPos.w) / _FoamThreshold);
float foamTex = tex2D(_FoamTexture, IN.worldPos.xz * _FoamTexture_ST.xy + _Time.y * float2(_FoamTextureSpeedX, _FoamTextureSpeedY));
float foam = step(foamDiff - (saturate(sin((foamDiff - _Time.y * _FoamLinesSpeed) * 8 * UNITY_PI)) * (1.0 - foamDiff)), foamTex);
```
***
## Water Normal
I mixed the water normal use one normal map scrolling in different directions. I blend the those two normals use an different ratio so that they looks more realistic
```c
inline float3 normal_strength_float(float3 inn,float str){
    return float3(inn.rg * str, lerp(1, inn.b, saturate(str)));
}

float2 uv1 = (IN.worldNormal.g*0.5+1)*IN.uv_NormalMap*_UVScale;
float2 uv2 = uv1 * _DetailScale;
float2 timescroll = float2(_Time.y,_Time.y)*_ScrollSpeed; 
float2 bias1 = float2(-0.1,0.035);
float2 bias2 = float2(-0.01,0.05);
uv1 += timescroll*bias1;
uv2 += timescroll*bias2;
float4 normalmap1 = tex2D(_NormalMap,uv1);
normalmap1.rgb = UnpackNormalmapRGorAG(normalmap1);
float4 normalmap2 = tex2D(_NormalMap,uv2);
normalmap2.rgb = UnpackNormalmapRGorAG(normalmap2);
float3 blend = normalize(float3(normalmap1.rg + normalmap2.rg, normalmap1.b * normalmap2.b));
float Ratio = 1 - _DetailStrength;
float3 normal = lerp(lerp(normalmap1.rgb, blend, saturate(Ratio*2)), normalmap2.rgb, saturate((Ratio-0.5)*2));
float strength = 1-saturate((1-IN.worldNormal.g)*0.5);
normal = normal_strength_float(normal,strength);
normal = normal_strength_float(normal,_BumpStrength);
float strength1 = saturate(IN.worldNormal.g+0.75);
normal = normal_strength_float(normal,strength1);
```
***
## Properties
| Name  |Description|
| ----  |   ----    |   
| _DepthColor | water depth color|
| _ShallowColor | water shallow color|
| _FoamThreshold | the foam size limits|
| _WaterDepth | the water depth level|
| _NormalMap  | the normal map |
| _UVScale | the uv scale of the normal map|
| _DetailScale | the blend ratio between detailed normal map|
| _ScrollSpeed | the normal scroll speed|
| _DetailStrength| the normal strength of lower normal|
| _BumpStrength | the bumping strength of higher normal|
| _FoamTexture | the foam texture |
| _FoamTextureSpeedX | the foam scroll speed x direction|
| _FoamTextureSpeedY | the foam scroll speed y direction|
| _FoamLinesSpeed | foam line speed|

***
## I am tired so I can hardly keep on 
later caustics and refractions and vertex wave

## Stylized water without normal (preview)
stylized water without normal
![Stylized water](./water/stylized_water.png)
stylized water with normal 
![Stylized water with normal](./water/water_withnormal.png)

[catlike-coding]: https://catlikecoding.com/unity/tutorials/flow/looking-through-water/
