---
layout: post
title: "Alpha分离"
date: 2016-10-31 00:00
comments: true
tags: 
	- Unity 
---

## 1. 为什么要进行Alpha分离？

* 使得图集可以压缩，减小包量。

## 2. Alpha分离要点

### 1. 分离

* 收集图集

  ```
  List<Texture2D> m_AtlasList = new List<Texture2D>();
  private void CollectAtlasSprite()
  {
      //Travel all AtlasSprite
      m_AtlasList.Add(obj as Texture2D);
  }  
  ```

<!-- more -->
* 分离单张图集的Alpha

* ```
  public static void separateOneTexture(Texture2D sourceTex)
  {
      string sourceTexPath= AssetDatabase.GetAssetPath(sourceTex);  

      //创建2张RGB格式的纹理，最后保存的时候会把rgba的alpha丢弃掉。
      Texture2D texRGB = new Texture2D(sourceTex.width, sourceTex.height, TextureFormat.RGB24, false);
      Texture2D texAlpha = new Texture2D(sourceTex.width, sourceTex.height, TextureFormat.RGB24, false);

      //取得原始像素数据
      Color[] srcColors = sourceTex.GetPixels();
      texRGB.SetPixels(srcColors);

      //生成 rgba = (a, a, a, a) 的图片
      Color[] alphaColors = new Color[srcColors.Length];
      for(int i=0;i<srcColors.Length;i++)
      {
          alphaColors[i].r = srcColors[i].a;
          alphaColors[i].g = srcColors[i].a;
          alphaColors[i].b = srcColors[i].a;
      }
      texAlpha.SetPixels(alphaColors);

      texRGB.Apply();
      texAlpha.Apply();

      byte[] rgbBytes = texRGB.EncodeToPNG();
      File.WriteAllBytes(sourceTexPath, rgbBytes);
      byte[] alphaBytes = texAlpha.EncodeToPNG();
      File.WriteAllBytes(alphaTexPath, alphaBytes);
  }
  ```

  > 记得检查图片的AssetImporter.GetAtPath\(sourceTexFullPath\).isReadable == true，否则不能进行SetPixels和GetPixels操作


### 2. 合并

* 合并Alpha图片和RGB图片

* ```

  public static void mergeTwoTexture(Texture2D t2dRGB,Texture2D t2dAlpha)
  {
      string strRGBPath = AssetDatabase.GetAssetPath(t2dRGB);
      string strAlphaPath = AssetDatabase.GetAssetPath(t2dAlpha);

      TextureImporter importerRGBTex = AssetImporter.GetAtPath(strRGBPath) as TextureImporter;
      if (!importerRGBTex.isReadable)
      {
          //检查isReadable属性
          importerRGBTex.isReadable = true;
          AssetDatabase.ImportAsset(strRGBPath);
      }

      TextureImporter importerAlphaTex = AssetImporter.GetAtPath(strAlphaPath) as TextureImporter;
      if(!importerAlphaTex.isReadable)
      {
          //检查isReadable属性
          importerAlphaTex.isReadable = true;
          AssetDatabase.ImportAsset(strAlphaPath);
      }

      Color[] rgbDatas = t2dRGB.GetPixels();
      Color[] alphaDatas = t2dAlpha.GetPixels();

      Color[] rgbaDatas = new Color[rgbDatas.Length];            
      Texture2D texRGBA = new Texture2D(t2dRGB.width, t2dRGB.height, TextureFormat.RGBA32,false);
      for(int nI=0;nI<rgbaDatas.Length;nI++)
      {
          rgbaDatas[nI].r = rgbDatas[nI].r;
          rgbaDatas[nI].g = rgbDatas[nI].g;
          rgbaDatas[nI].b = rgbDatas[nI].b;
          rgbaDatas[nI].a = alphaDatas[nI].r;
      }

      texRGBA.SetPixels(rgbaDatas);
      texRGBA.Apply();

      byte[] rgbaBytes = texRGBA.EncodeToPNG();
      File.WriteAllBytes(strRGBPath, rgbaBytes);
      AssetDatabase.DeleteAsset(strAlphaPath);
      AssetDatabase.Refresh();
      ReImportAsset(strRGBPath);         
  }
  ```


