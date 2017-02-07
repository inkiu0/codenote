---
layout: post
title: "顶点动画"
date: 2017-01-11 00:00
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/UVAnimation.gif)  
![](/assets/blogImg/UnityShader/UVAnimation2.gif)  
对顶点进行位移以使得图片看起来像动画  
- _Frequency 是震动频率
- _InvWaveLength 是波长的倒数
- _Magnitude 是振幅  
<!-- more -->
```  
Shader "UnityShaderLearning/UVAnimation"
{
    Properties
    {
        _MainTex ("Main Tex", 2D) = "white" {}
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _Magnitude("Distortion Magnitude", Float) = 1
        _Frequency ("Distortion Frequency", Float) = 1
        _InvWaveLength ("Distortion Inverse Wave Length", Float) = 10
        _Speed ("Speed", Float) = 0.5
    }
    SubShader
    {
        Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True" }

        Pass
        {
            Tags { "LightMode"="ForwardBase" }

            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            Cull Off
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;
            float _Magnitude;
            float _Frequency;
            float _InvWaveLength;
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
                float4 offset;
                offset.yzw = float3(0.0, 0.0, 0.0);
                offset.x = sin(_Frequency * _Time.y + v.vertex.x * _InvWaveLength + v.vertex.y * _InvWaveLength + v.vertex.z * _InvWaveLength) * _Magnitude;
                o.pos = mul ( UNITY_MATRIX_MVP, v.vertex + offset);
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                o.uv += float2(0.0, _Time.y * _Speed);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                fixed4 c = tex2D(_MainTex, i.uv);
                c.rgb *= _Color.rgb;
                return c;
            }
            ENDCG
        }
    }
    Fallback "Transparent/VertexLit"
}
```  