---
layout: post
title: "Sobel算子边缘检测"
date: 2017-01-17 00:00
comments: true
tags: 
	- UnityShader 
---
用Sobel算子来进行边缘检测  
应该知道除了Sobel算子以外还有几种常用的算子。
- Roberts  

$$\left[\begin{matrix}
-1 & 0 \\
 0 & 1
\end{matrix}\right]$$$$\left[\begin{matrix}
0 & -1 \\
1 &  0
\end{matrix}\right]$$

- Prewitt

$$\left[\begin{matrix}
-1 & -1 & -1 \\
 0 &  0 &  0 \\
 1 &  1 &  1
\end{matrix}\right]$$$$\left[\begin{matrix}
-1 &  0 &  1 \\
-1 &  0 &  1 \\
-1 &  0 &  1
\end{matrix}\right]$$


- Sobel

$$\left[\begin{matrix}
-1 & -2 & -1 \\
0 & 0 & 0 \\
1 & 2 & 1
\end{matrix}\right]$$$$\left[\begin{matrix}
-1 & 0 & 1 \\
-2 & 0 & 2 \\
-1 & 0 & 1
\end{matrix}\right]$$

我们取得一个纹素的大小， 即1/texelSize。然后根据现在的纹理坐标前后左右偏移一个像素，得到9个方向的纹理坐标。  
然后用Sobel函数算得该纹理坐标的卷积值  
- _EdgeOnly = 0时显示原图像和边缘的叠加图像，即withEdgeColor
- _EdgeOnly = 1时只显示边缘，其他部分为背景色，即只显示onlyEdgeColor
<!-- more -->
```  
v2f vert(appdata_img v)
			{
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				half2 uv = v.texcoord;

				o.uv[0] = uv + _MainTex_TexelSize.xy * half2(-1, -1);
				o.uv[1] = uv + _MainTex_TexelSize.xy * half2(0, -1);
				o.uv[2] = uv + _MainTex_TexelSize.xy * half2(1, -1);
				o.uv[3] = uv + _MainTex_TexelSize.xy * half2(-1, 0);
				o.uv[4] = uv + _MainTex_TexelSize.xy * half2(0, 0);
				o.uv[5] = uv + _MainTex_TexelSize.xy * half2(1, 0);
				o.uv[6] = uv + _MainTex_TexelSize.xy * half2(-1, 1);
				o.uv[7] = uv + _MainTex_TexelSize.xy * half2(0, 1);
				o.uv[8] = uv + _MainTex_TexelSize.xy * half2(1, 1);

				return o;
			}

			fixed luminance(fixed4 color)
			{
				return 0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b;
			}

			half Sobel(v2f i)
			{
				const half Gx[9] = { -1, -2, -1,
					0,  0,  0,
					1,  2,  1 };
				const half Gy[9] = { -1,  0,  1,
					-2,  0,  2,
					-1,  0,  1 };
				half texColor;
				half edgeX = 0;
				half edgeY = 0;
				for (int it = 0; it < 9; it++)
				{
					texColor = luminance(tex2D(_MainTex, i.uv[it]));
					edgeX += texColor * Gx[it];
					edgeY += texColor * Gy[it];
				}
				half edge = 1 - abs(edgeX) - abs(edgeY);
				return edge;
			}

			fixed4 frag(v2f i) : SV_Target
			{
				half edge = Sobel(i);
				fixed4 withEdgeColor = lerp(_EdgeColor, tex2D(_MainTex, i.uv[4]), edge);
				fixed4 onlyEdgeColor = lerp(_EdgeColor, _BackgroundColor, edge);
				return lerp(withEdgeColor, onlyEdgeColor, _EdgeOnly);
			}
```  