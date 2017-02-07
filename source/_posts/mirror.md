---
layout: post
title: "镜子效果"
date: 2016-12-28 00:43
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/QQ图片20161228004324.png)  

一个简单的shader就能实现镜子的效果。在镜子的位置放一个相机，把相机的RenderTexture翻转，1 - uv.x。  
<!-- more -->
```  
Shader "UnityShaderLearning/Mirror"
{
    Properties
    {
        _MainTex ("Main Tex", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry" }
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            sampler2D _MainTex;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
                o.uv = v.texcoord;
                o.uv.x = 1 - o.uv.x;
                return o;
            }

            fixed4 frag(v2f i):SV_Target
            {
                return tex2D(_MainTex, i.uv);
            }
            ENDCG
        }
    }
 	FallBack Off
}
```  
