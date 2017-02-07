---
layout: post
title: "序列帧动画"
date: 2017-01-09 00:00
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/ImageSequence.gif)  

用shader对序列帧图片进行采样，以实现动画效果。
<!-- more -->
```  
Shader "UnityShaderLearning/ImageSequenceAni"
{
    Properties
    {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Image Sequence", 2D) = "white"{}
        _HorizontalAmount ("Horizontal Amount", Float) = 4
        _VertcalAmount ("Vertcal Amount", Float) = 4
        _Speed ("Speed", Range(1, 100)) = 30
    }
    SubShader
    {
        Tags { "Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" }

        Pass
        {
            Tags { "LightMode"="ForwardBase" }

            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            float _HorizontalAmount;
            float _VertcalAmount;
            float _Speed;

            struct a2v
            {
                float4 vertex : POSITION;
                float2 texcoord : TEXCOORD0;
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
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                float time = floor(_Time.y * _Speed);
                float row = floor(time / _HorizontalAmount);
                float col = floor(time / _VertcalAmount);

                half2 uv = i.uv + half2(col, -row);
                uv.x /= _HorizontalAmount;
                uv.y /= _VertcalAmount;

                fixed4 color = tex2D(_MainTex, uv);
                color.rgb *= _Color;
                return color;
            }

            ENDCG
        }
    }
	FallBack "Transparent/VertexLit"
}
```   