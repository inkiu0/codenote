---
layout: post
title: "切线空间下的法线贴图应用"
date: 2016-12-07 00:26
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/QQ图片20161207002627.png)
- 左侧BumpScale为-1
- 右侧BumpScale为1
<!-- more -->
```
Shader "UnityShaderLearning/NormalMapTangentSpace"
{
	Properties
	{
		_Color("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex("Main Tex", 2D) = "white"{}
		_BumpMap("Normal Map", 2D) = "bump"{}
		_BumpScale("Bump Scale", Float) = 1.0
		_Specular("Specular", Color) = (1, 1, 1, 1)
		_Gloss("Gloss", Range(8.0, 256)) = 20
	}
	SubShader
	{
		Pass
		{
			Tags { "LightMode"="ForwardBase" }

			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#include "Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;
			fixed4 _Specular;
			float _Gloss;

			struct a2v
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 tangent : TANGENT;
				float4 texcoord : TEXCOORD0;
			};

			struct v2f
			{
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
				float3 LightDir : TEXCOORD1;
				float3 ViewDir : TEXCOORD2;
			};

			v2f vert(a2v v)
			{
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				o.uv.xy = v.vertex.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				o.uv.zw = v.vertex.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;
				float3 binormal = cross(normalize(v.normal), normalize(v.tangent.xyz));
				float3x3 rotation = float3x3(v.tangent.xyz, binormal, v.normal);
				o.LightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
				o.ViewDir = mul(rotation, ObjSpaceViewDir(v.vertex)).xyz;

				return o;
			}

			fixed4 frag(v2f i) : SV_Target
			{
				fixed3 tangentLightDir = normalize(i.LightDir);
				fixed3 tangentViewDir = normalize(i.ViewDir);
				fixed4 packedNormal = tex2D(_BumpMap, i.uv.zw);
				fixed3 tangentNormal;
				// tangentNormal.xy = (packedNormal.xy * 2 - 1) * _BumpScale;
				// tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

				tangentNormal = UnpackNormal(packedNormal);
				tangentNormal.xy *= _BumpScale;
				tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT * albedo;

				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));
				fixed3 halfDir = normalize(tangentViewDir + tangentLightDir);
				fixed3 specular = _LightColor0 * _Specular.rgb * pow(max(0, dot(tangentNormal, halfDir)), _Gloss);

				return fixed4(ambient + diffuse + specular, 1.0);
			}

			ENDCG
		}
	}
	FallBack "specular"
}
```