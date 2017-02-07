---
layout: post
title: "DiffuseVertexLevel/PixelLevel(漫反射)"
date: 2016-11-18 00:24
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/QQ图片20161118002425.png)
左为逐顶点漫反射光照模型，右为逐像素漫反射光照模型。  

漫反射光照模型
$$C_{diffuse}=(C_{light} * m_{diffuse})max(0, \vec{n}·\vec{i})$$
- $$C_{light}$$入射光颜色和强度
- $$m_{diffuse}$$材质漫反射系数
- $$\vec{n}$$法线方向
- $$\vec{i}$$光源方向  

<!-- more -->
```
Shader "UnityShaderLearning/DiffuseVertexLevel"
{
    Properties
    {
        _Diffuse("Diffuse", Color) = (1, 1, 1, 1)
    }
    SubShader
    {
        Pass
        {
            Tags{ "LightMode"="ForwardBase" }
            
            CGPROGRAM
            
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Diffuse;
            
            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };
    
            struct v2f
            {
                float4 pos : SV_POSITION;
                fixed3 color : COLOR;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal = normalize(mul(v.normal, (float3x3)_World2Object));
                fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));
                o.color = ambient + diffuse;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                return fixed4(i.color, 1.0);    
            }
            ENDCG
        }
    }
    FallBack "Diffuse"
}
```
```
Shader "UnityShaderLearning/DiffusePixelLevel"
{
	Properties
	{
		_Diffuse("Diffuse", Color) = (1, 1, 1, 1)
	}
	SubShader
	{
		Pass
		{
			Tags{ "LightMode" = "ForwardBase" }

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag
			#include "Lighting.cginc"

			fixed4 _Diffuse;

			struct a2v
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f
			{
				float4 pos : SV_POSITION;
				fixed3 worldNormal : TEXCOORD0;
			};

			v2f vert(a2v v)
			{
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				o.worldNormal =mul(v.normal, (float3x3)_World2Object);
				return o;
			}

			fixed4 frag(v2f i) : SV_Target
			{
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
				fixed3 color = ambient + diffuse;
				return fixed4(color, 1.0);
			}
			ENDCG
		}
	}
	FallBack "Diffuse"
}
```