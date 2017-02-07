---
layout: post
title: "bloom效果"
date: 2017-01-25 00:00
comments: true
tags: 
	- UnityShader 
---
bloom是模拟真实相机，较亮的部分扩散到周围，造成一种朦胧的感觉。  
* 先用第一个pass筛选出原图中较亮的部分  
* 然后在第二和第三个pass中把较亮的图像进行高斯模糊  
* 最后在第四个pass中把模糊后的亮部和原图进行混合得到bloom效果  

```  
    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            material.SetFloat("_LuminanceThreshold", luminanceThreshold);
            int rtW = src.width / downSample;
            int rtH = src.height / downSample;
            RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
            buffer0.filterMode = FilterMode.Bilinear;
            Graphics.Blit(src, buffer0, material, 0);
            for (int i = 0; i < iterations; i++)
            {
                material.SetFloat("_BlurSize", 1.0f + i * blurSpread);
                RenderTexture buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);
                Graphics.Blit(buffer0, buffer1, material, 1);
                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
            }
            material.SetTexture("_Bloom", buffer0);
            Graphics.Blit(src, dest, material, 3);
            RenderTexture.ReleaseTemporary(buffer0);
        }
        else
        {
            Graphics.Blit(src, dest);
        }
    }
```  
<!-- more -->
```  
Shader "UnityShaderLearning/Bloom"
{
	Properties
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_Bloom ("Bloom (RGB)", 2D) = "black" {}
		_LuminanceThreshold("Luminance Threshold", Float) = 0.5
		_BlurSize("Blur Size", Float) = 1.0
	}
	SubShader
	{
		CGINCLUDE
		#include "UnityCG.cginc"
		sampler2D _MainTex;
		half4 _MainTex_TexelSize;
		sampler2D _Bloom;
		float _LuminanceThreshold;
		float _BlurSize;

		struct v2f
		{
			float4 pos : SV_POSITION;
			half2 uv : TEXCOORD0;
		};

		v2f vertExtractBright(appdata_img v)
		{
			v2f o;
			o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
			o.uv = v.texcoord;
			return o;
		}

		fixed luminance(fixed4 color)
		{
			return 0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b;
		}

		fixed4 fragExtractBright(v2f i) : SV_Target
		{
			fixed4 c = tex2D(_MainTex, i.uv);
			fixed val = clamp(luminance(c) - _LuminanceThreshold, 0.0, 1.0);
			return c * val;
		}

		struct v2fBloom
		{
			float4 pos : SV_POSITION;
			half4 uv : TEXCOORD0;
		};

		v2fBloom vertBloom(appdata_img v)
		{
			v2fBloom o;
			o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
			o.uv.xy = v.texcoord;
			o.uv.zw = v.texcoord;
			#if UNITY_UV_STARTS_AT_TOP
			if (_MainTex_TexelSize.y < 0.0)
				o.uv.w = 1.0 - o.uv.w;
			#endif
			return o;
		}

		fixed4 fragBloom(v2fBloom i) : SV_Target
		{
			return tex2D(_MainTex, i.uv.xy) * tex2D(_Bloom, i.uv.zw);
		}
		ENDCG
		ZTest Always Cull Off ZWrite Off
		Pass
		{
			CGPROGRAM
			#pragma vertex vertBloom
			#pragma fragment fragBloom
			ENDCG
		}

		UsePass "UnityShaderLearning/GaussianBlurShader/GAUSSIAN_BLUR_VERTICAL"

		UsePass "UnityShaderLearning/GaussianBlurShader/GAUSSIAN_BLUR_HORIZONTAL"

		Pass
		{
			CGPROGRAM
			#pragma vertex vertBloom
			#pragma fragment fragBloom
			ENDCG
		}
	}
	Fallback Off
}
```  