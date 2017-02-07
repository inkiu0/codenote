---
layout: post
title: "运动模糊 "
date: 2017-01-26 00:00
comments: true
tags: 
	- UnityShader 
---
#MotionBlur运动模糊  
- 首先为accumulationTexture进行一个恢复操作，以取得上一步混合得到的纹理。  
- 然后用shader把原图像和上一步的纹理进行混合  
- 最后把混合后的纹理显示在屏幕上。  

```  
accumulationTexture.MarkRestoreExpected();
material.SetFloat("_BlurAmount", 1.0f - blurAmount);
Graphics.Blit(src, accumulationTexture, material);
Graphics.Blit(accumulationTexture, dest);
```  
shader混合图像的方式为(last.rgb * BlurAmount + src.rgb * (1 - blurAmount), last.a)  
<!-- more -->
```  
Shader "UnityShaderLearning/MotionBlur"
{
	Properties
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_BlurAmount("Blur Amount", FLoat) = 1.0
	}
	SubShader
	{
		CGINCLUDE
		#include "UnityCG.cginc"
		sampler2D _MainTex;
		fixed _BlurAmount;
		struct v2f
		{
			float4 pos : SV_POSITION;
			half2 uv : TEXCOORD0;
		};
		v2f vert(appdata_img v)
		{
			v2f o;
			o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
			o.uv = v.texcoord;
			return o;
		}
		fixed fragRGB(v2f i) : SV_Target
		{
			return fixed4(tex2D(_MainTex, i.uv).rgb, _BlurAmount);
		}
			half4 fragA(v2f i) : SV_Target
		{
			return tex2D(_MainTex, i.uv);
		}
		ENDCG
		ZTest Always Cull Off ZWrite Off
		Pass
		{
			Blend SrcAlpha OneMinusSrcAlpha
			ColorMask RGB
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment fragRGB
			ENDCG
		}
		Pass
		{
			Blend One Zero
			ColorMask A
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment fragA
			ENDCG
		}
	}
Fallback Off
}
```  