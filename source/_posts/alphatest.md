---
layout: post
title: "透明测试"
date: 2017-01-16 00:00
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/QQ图片20161209002905.png)  

- AlphaTest  
上图为简单的AlphaTest，Alpha值小于0.55的颜色抛弃。判断的语法`clip (texcoordColor.a - value)`也可写作  
<!-- more -->  
```  
if((texcoordColor.a - value) < 0.0)
{
    discard;
}  
```  

- 双面渲染  
AlphaTest开启双面渲染也比较容易，只需要在`Tags={ "LightMode"="ForwardBase" }`后添加一行`Cull Off`, 关闭剔除即可开启双面渲染。

# 2. 透明混合(AlphaBlend)
![](/assets/blogImg/UnityShader/QQ图片20161209005804.png)![](/assets/blogImg/UnityShader/QQ图片20161209005808.png)  
- AlphaBlend  
在Pass中开启Blend即可开启透明混合，混合的公式为`混合颜色 = 当前物体Alpha * 当前颜色 + (1 - 当前Alpha) * 背景颜色`  
- AlphaBlendZWrite  
AlphaBlend的shader添加一个Pass即可开启深度写入  
```
Pass
{
ZWrite On
ColorMask 0
}
```  
![](/assets/blogImg/UnityShader/QQ图片20161211225510.png)  
`ColorMask 0`即关闭该Pass的颜色通道写入，即该Pass只执行了深度写入，然后在后续Pass中处理其他操作。开启ZWrite以后相比AlphaBlend关闭ZWrite能更好地体现前后关系。  
- 双面渲染  
![](/assets/blogImg/UnityShader/QQ图片20161212001732.png)  
AlphaBlend的双面渲染要比AlphaTest的稍微复杂一些, 复制原来AlphaBlend的代码，写作2个Pass，分别剔除Front渲染背面和剔除Back渲染前面。  
```  
Pass
{
    Cull Front
    //原来AlphaBlend的代码
}
Pass
{
    Cull Back
    //原来AlphaBlend的代码
}  
```  
  
- Photoshop中AlphaBlend的几种常用模式  
>//正常(Normal)，透明度混合    
Blend SrcAlpha OneMinusSrcAlpha  
>
>//柔和相加(Soft Additive)  
Blend OneMinusDstColor One  
>
>//正片叠底(multiply 相乘)  
Blend DstColor One  
>
>//变暗(Darken)  
BlendOp Min  
Blend One One  
>
>//变亮(Lighten)  
BlendOp Max  
Blend One One  
>
>//滤色(Screen)  
Blend OneMinusDstColor One  
>>//等同于Blend One OneMinusSrcColor  
//等于D + S - D * S  
>
>线性减淡(Linear Dodge)  
Blend One One  

```   
Shader "UnityShaderLearning/AlphaTest" 
{
	Properties
	{
		_Color ("Main Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Main Tex", 2D) = "white"{}
		_Cutoff ("Alpha Cutoff", Range(0, 1)) = 0.55
	}
	SubShader
	{
		Tags { "Queue"="AlphaTest" "IgnoreProjector" = "True" "RenderType"="TransparentCutout"}

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
			fixed _Cutoff;

			struct a2v
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 texcoord : TEXCOORD0;
			};

			struct v2f
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				float2 uv : TEXCOORD2;
			};

			v2f vert(a2v v)
			{
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(_Object2World, v.vertex).xyz;
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				return o;
			}

			fixed4 frag(v2f i) : SV_Target
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed4 texColor = tex2D(_MainTex, i.uv);
				clip (texColor.a - _Cutoff);

				fixed3 albedo = texColor.rgb * _Color.rgb;
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir));
				return fixed4(ambient + diffuse, 1.0);
			}
			ENDCG
		}
	}
	FallBack "Transparent/Cutout/VertexLit"
}  
```  

```   
Shader "UnityShaderLearning/AlphaBlend"
{
    Properties
    {
        _Color ("Main Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Main Tex", 2D) = "white"{}
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 1
    }

	SubShader
	{
		Tags { "Queue"="AlphaTest" "IgnoreProjector"="True" "RenderType"="TransparentCutout"}

		Pass
		{
			Tags { "LightMode"="ForwardBase" }
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag
			#include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _AlphaScale;

			struct a2v
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 texcoord : TEXCOORD0;
			};

			struct v2f
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				float2 uv : TEXCOORD2;
			};

			v2f vert(a2v v)
			{
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(_Object2World, v.vertex).xyz;
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				return o;
			}

			fixed4 frag(v2f i) : SV_Target
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed4 texColor = tex2D(_MainTex, i.uv);
				fixed3 albedo = texColor.rgb * _Color.rgb;
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir));
				return fixed4(ambient + diffuse, texColor.a * _AlphaScale);
			}
			ENDCG
		}
	}
	FallBack "Transparent/VertexLit"
}  
```  