---
layout: post
title: "亮度饱和度对比度屏幕后处理效果"
date: 2017-01-16 00:00
comments: true
tags: 
	- UnityShader 
---
以下是调节屏幕亮度饱和度对比度的关键代码，cs脚本挂在Camera上。
```  
//给shader传对应的亮度/饱和度/对比度
void OnRenderImage(RenderTexture src, RenderTexture dest)
{
    if(material != null)
    {
        material.SetFloat("_Brightness", brightness);
        material.SetFloat("_Saturation", saturation);
        material.SetFloat("_Contrast", contrast);

        Graphics.Blit(src, dest, material);
    }
    else
    {
        Graphics.Blit(src, dest);
    }
}
```  
<!-- more -->
```  
//fragment shader用csharp传来的值进行插值得到屏幕处理后的效果
fixed4 frag(v2f i) : SV_Target
{
    fixed4 renderTex = tex2D(_MainTex, i.uv);
    fixed3 finalColor = renderTex.rgb * _Brightness;
    fixed luminance = 0.2125 * renderTex.r + 0.7154 * renderTex.g + 0.0721 * renderTex.b;
    fixed3 luminanceColor = fixed3(luminance, luminance, luminance);
    finalColor = lerp(luminanceColor, finalColor, _Saturation);

    fixed3 avgColor = fixed3(0.5, 0.5, 0.5);
    finalColor = lerp(avgColor, finalColor, _Contrast);
    return fixed4(finalColor, renderTex.a);
}
```  