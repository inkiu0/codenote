---
layout: post
title: "广告牌"
date: 2017-01-14 00:00
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/BillBoarding.gif)  
我们根据设置的法线y方向偏移来得到偏移后的法线，然后和upDir叉乘得到右方向，最后用法线和右方向叉乘得到准确的upDir。  
最后把顶点坐标通过我们构造的矩阵进行变化，从而得到新的顶点坐标，最后转换到观察空间。我们就可以得到始终面向我们的物体。  
<!-- more -->
```  
Shader "UnityShaderlearning/BillBoarding"
{
    Properties
    {
        _MainTex ("Main Tex", 2D) = "white" {}
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _VerticalBillboarding ("Vertical Restraints", Range(0, 1)) = 1
    }
    SubShader
    {
        Tags { "Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True" }

        Pass
        {
            Tags { "LightMode"="ForwardBase" }

            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            Cull Off

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;
            fixed _VerticalBillboarding;

            struct a2v
            {
                float4 vertex : POSITION;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            v2f vert(a2v v)
            {
                v2f o;
                float3 center = float3(0, 0, 0)
                float3 viewer = mul(_World2Object, float4(_WorldSpaceCameraPos, 1));
                float3 normalDir = viewer - center;
                normalDir.y = normalDir.y * _VerticalBillboarding;
                normalDir = normalize(normalDir);
                float3 upDir = abs(normalDir.y) > 0.99 ? float3(0, 0, 1) : float3(0, 1, 0);
                float3 rightDir = normalize(cross(upDir, normalDir));
                upDir = normalize(cross(normalDir, rightDir));
                floar3 centerOffs = v.vertex.xyz - center;
                floar3 localPos = center + rightDir * centerOffs.x + upDir * centerOffs.y = normalDir * centerOffs.z;
                o.pos = mul(UNITY_MATRIX_MVP, float4(localPos, 1));
				o.uv = TRANSFORM_TEX(v.texcoord,_MainTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                fixed4 c = tex2D (_MainTex, i.uv);
                c.rgb *= _Color.rgb;
                return c;
            }
            ENDCG
        }
    }
    Fallback "Transparent/VertexLit"
}
```  