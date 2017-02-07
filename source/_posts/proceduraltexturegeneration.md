---
layout: post
title: "程序纹理"
date: 2017-01-06 00:00
comments: true
tags: 
	- UnityShader 
---
![](/assets/blogImg/UnityShader/proceduralTexture.png)  

性能消耗以秒计...不实用。  
<!-- more -->
```  
    private Texture2D GenerateProceduralTexture()
    {
        Texture2D pTex = new Texture2D(textureWidth, textureWidth);
        float circleInterval = textureWidth / 4.0f;
        float radius = textureWidth / 10.0f;
        float edgeBlur = 1.0f / blurFactor;

        for(int w = 0; w < textureWidth; w++)
        {
            for(int h = 0; h < textureWidth; h++)
            {
                Color pixel = backgroundColor;
                for (int i = 0; i < 3; i++)
                {
                    for(int j = 0; j < 3; j++)
                    {
                        Vector2 circleCenter = new Vector2(circleInterval * (i + 1), circleInterval * (j + 1));
                        float dist = Vector2.Distance(new Vector2(w, h), circleCenter) - radius;
                        Color color = MixColor(circleColor, new Color(pixel.r, pixel.g, pixel.b, 0.0f), Mathf.SmoothStep(0f, 1.0f, dist * edgeBlur));
                        pixel = MixColor(pixel, color, color.a);
                    }
                }
                pTex.SetPixel(w, h, pixel);
            }
        }
        pTex.Apply();
        return pTex;
    }

    private Color MixColor(Color color0, Color color1, float mixFactor)
    {
        Color mixColor = Color.white;
        mixColor.r = Mathf.Lerp(color0.r, color1.r, mixFactor);
        mixColor.g = Mathf.Lerp(color0.g, color1.g, mixFactor);
        mixColor.b = Mathf.Lerp(color0.b, color1.b, mixFactor);
        mixColor.a = Mathf.Lerp(color0.a, color1.a, mixFactor);
        return mixColor;
    }
```  
性能消耗以秒计...不实用。