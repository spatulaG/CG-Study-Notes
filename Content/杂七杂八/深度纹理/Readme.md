Unity里的转换代码（不好
```
UnityCG.cginc

// Encoding/decoding [0..1) floats into 8 bit/channel RGBA. Note that 1.0 will not  be encoded properly.
inline float4 EncodeFloatRGBA( float v )
{
    float4 kEncodeMul = float4(1.0, 255.0, 65025.0, 16581375.0);//255^0, 255^1, 255^2, 255^3
    float kEncodeBit = 1.0/255.0;
    float4 enc = kEncodeMul * v;
    enc = frac (enc);
    enc -= enc.yzww * kEncodeBit;
    return enc;
}
inline float DecodeFloatRGBA( float4 enc )
{
    float4 kDecodeDot = float4(1.0, 1/255.0, 1/65025.0, 1/16581375.0);
    return dot( enc, kDecodeDot );
}
```

做的就是：

```
inline float4 EncodeFloatRGBA(float v)
{
    byte[] eb = BitConverter.GetBytes(v);
    if (BitConverter.IsLittleEndian)//倒序
    {
        return float4(eb[3] / 255.0f, eb[2] / 255.0f, eb[1] / 255.0f, eb[0] / 255.0f);
    }

    return float4(eb[0] / 255.0f, eb[1] / 255.0f, eb[2] / 255.0f, eb[3] / 255.0f);
}
```

Linear01Depth会返回View空间中范围在(0，1]的深度，近平面为Near/Far，远平面为1。

LinearEyeDepth会返回View空间中的深度，近平面为Near，远平面为Far。

```
// Z buffer to linear 0..1 depth
inline float Linear01Depth( float z )
{
    return 1.0 / (_ZBufferParams.x * z + _ZBufferParams.y);
}
// Z buffer to linear depth
inline float LinearEyeDepth( float z )
{
    return 1.0 / (_ZBufferParams.z * z + _ZBufferParams.w);
}
```

NDC的Z是[-w,w], 


采样得到的深度值范围是0 ~ 1，NDC空间下深度值范围是-1 ~ 1，所以有：

![image](https://user-images.githubusercontent.com/29577919/169681794-0804dea3-f8ff-4fde-9fe7-0d2d6b5c28cd.png)

NDC坐标是裁剪空间坐标经过齐次除法得到的，所以有：

![image](https://user-images.githubusercontent.com/29577919/169681798-217ae5c5-688f-4131-a41b-52ca0bdb3881.png)

定义Far为远平面距离，Near为近平面距离，根据投影变换，可以得到Zclip和Zview的计算公式：[推导在这里](https://github.com/spatulaG/CG-Study-Notes/blob/main/Content/%E6%9D%82%E4%B8%83%E6%9D%82%E5%85%AB/%E6%B7%B1%E5%BA%A6%E7%BA%B9%E7%90%86/Readme.md#zview-%E5%88%B0-zclip%E7%9A%84%E6%8E%A8%E5%AF%BC)

![image](https://user-images.githubusercontent.com/29577919/169681800-21abe9d8-5251-467b-9436-3af1019b5399.png)

![image](https://user-images.githubusercontent.com/29577919/169681801-72fab073-fd23-4f77-9b77-e284c020806e.png)

所以反过来可以得出Zview和Zclip的关系：

![image](https://user-images.githubusercontent.com/29577919/169681803-749a4010-99fb-4727-89eb-85811bd796bc.png)

代入Zclip和Zndc，可以求出：

![image](https://user-images.githubusercontent.com/29577919/169681804-f57ad9a6-b4f4-4a05-8ddb-50014638fd83.png)

Unity的观察空间是右手坐标系，这里求出的Zview是负值，Linear01Depth和LinearEyeDepth会返回正值。

## 重建世界坐标
首先求出NDC坐标，xy通过VS输出的齐次坐标求出，z通过采样深度纹理得出：
```
// VS
o.screenPos = ComputeScreenPos(o.vertex);

// FS
float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
float4 ndc = float4(i.screenPos.x / i.screenPos.w, i.screenPos.y / i.screenPos.w, depth, 1);
```
再经过View、Projection的逆变换就能得到世界坐标：
```
// FS
float4 worldPosHomog = mul(_InverseViewProjectionMatrix, ndc);
float4 worldPos = worldPosHomog / worldPosHomog.w;
```



## Zview 到 Zclip的推导

![image](https://user-images.githubusercontent.com/29577919/169683913-2f170b91-5b65-4000-9363-4e42d376c60d.png)
![image](https://user-images.githubusercontent.com/29577919/169683916-f69bf0a1-e2bb-4f5e-83b1-c4387886383e.png)
### 横轴的推导
![image](https://user-images.githubusercontent.com/29577919/169683949-5864d77f-9270-4716-b599-1f38dab1eccd.png)
![image](https://user-images.githubusercontent.com/29577919/169683946-d3368dfe-19af-4102-8db2-0d4c5471313e.png)
### 同理， 由此我们已经有了xy的矩阵
![image](https://user-images.githubusercontent.com/29577919/169683972-c710fa5f-1f91-4edb-be84-153daa4a3e11.png)
### Z的推导稍微不一样一点因为Z是n到f之间
![image](https://user-images.githubusercontent.com/29577919/169683981-531c9ce3-07fb-4e07-9846-b05fc1059f20.png)
![image](https://user-images.githubusercontent.com/29577919/169683991-b9a65e7c-38fa-4355-8db3-1f472a2fc718.png)
![image](https://user-images.githubusercontent.com/29577919/169683995-f5079afe-1709-4706-a315-4bc9479e96cc.png)
### 那么最终矩阵是
![image](https://user-images.githubusercontent.com/29577919/169684003-2af79d2b-e30f-416a-8e93-e829e21274b2.png)

## 引用
[opengl 透视投影矩阵的推导](https://www.scratchapixel.com/lessons/3d-basic-rendering/perspective-and-orthographic-projection-matrix/opengl-perspective-projection-matrix)
