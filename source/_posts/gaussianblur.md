---
layout: post
title: "高斯模糊"
date: 2017-01-22 00:00
comments: true
tags: 
	- UnityShader 
---
首先创建了一块1/downSample屏幕大小的buffer存入当前屏幕图像，然后设置滤波模式为双线性采样。  
这样需要处理的像素是原来的1/downSample。  
然后多次进行高斯模糊，逐次增大_BlurSize。  
<!-- more -->
```  
    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if(material != null)
        {
            int rtW = src.width;
            int rtH = src.height;
            RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
            buffer0.filterMode = FilterMode.Bilinear;
            Graphics.Blit(src, buffer0);
            for(int i = 0; i < iterations; i++)
            {
                material.SetFloat("_BlurSize", 1.0f + i * blurSpread);
                RenderTexture buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);
                Graphics.Blit(buffer0, buffer1, material, 0);
                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
                buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);
                Graphics.Blit(buffer0, buffer1, material, 1);
                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
            }
            Graphics.Blit(buffer0, dest);
            RenderTexture.ReleaseTemporary(buffer0);
        }
        else
        {
            Graphics.Blit(src, dest);
        }
    }
```  
这个shader的顶点着色器用来分别计算横竖2组像素位置，片元着色器是相同的。  
因为高斯模糊的卷积核权重是对称的，所以可以只用一个长度为3的数组描述。  
顶点shader输入的uv坐标都是从当前点平移1-2个纹素得到的，比如得到当前点右边相邻的这个格子uv + (texelSiz * 1, 0)  
```  
		fixed4 fragBlur(v2f i) : SV_Target
		{
			float weight[3] = {0.4026, 0.2442, 0.0545};
			fixed3 sum = tex2D(_MainTex, i.uv[0]).rgb * weight[0];
			for (int it = 1; it < 3; it++)
			{
				sum += tex2D(_MainTex, i.uv[it]).rgb * weight[it];
				sum += tex2D(_MainTex, i.uv[2 * it]).rgb * weight[it];
			}
			return fixed4(sum, 1.0);
		}
```  