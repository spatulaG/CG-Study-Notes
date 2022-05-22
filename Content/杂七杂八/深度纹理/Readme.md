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

## 1.重建世界坐标
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

NDC和Clip坐标的关系：
![image](https://user-images.githubusercontent.com/29577919/169684283-3ad895a6-9aa4-442d-99d1-d3997ebf2705.png)

World和Clip的关系:
![image](https://user-images.githubusercontent.com/29577919/169684279-50316ef7-f308-40cd-a7e5-a336a6ba0850.png)

所以
![image](https://user-images.githubusercontent.com/29577919/169684288-996f7ed5-8dbd-4e1b-ba5c-5b692c17ac5e.png)


![image](https://user-images.githubusercontent.com/29577919/169684342-ef6739a9-4e47-4d67-9f33-1a507c163b8d.png)
所以世界坐标就等于NDC坐标进行VP逆变换之后再除以w。

## 2.射线法
![image](https://user-images.githubusercontent.com/29577919/169684451-e8e51265-4e9e-4ebd-859f-9674cc8e7a21.png)
根据Camera的信息，求出世界空间中Camera到远平面四个角的向量
根据Camera的fov和farPlane求出远平面的宽高的一半（这里以透视相机为例）：

```
float halfFarPlaneHeight = m_Camera.farClipPlane * Mathf.Tan(m_Camera.fieldOfView  / 2 * Mathf.Deg2Rad);
float halfFarPlaneWidth = halfFarPlaneHeight * m_Camera.aspect;
如果所示，相机到远平面右上角的向量topRight = C_F + F_R + R_TR
```

```
Vector4 CF = m_Camera.transform.forward * m_Camera.farClipPlane;
Vector4 R_TR = m_Camera.transform.up * halfFarPlaneHeight;
Vector4 FR = m_Camera.transform.right * halfFarPlaneWidth;

Vector4 topRight = CF + FR + R_TR;
```

依次求出四个角，传入到Shader中，进行类似后处理时，屏幕的四个点对应远平面的四个点，在Shader中根据UV算出index，获取对应的向量：
```
// VS
index = v.uv.x + (2 * v.uv.y);
o.farPlaneVector = _FarPlaneVector[index];
```
经过插值输入到PS中就是相机摄像当前像素的向量CG（起点是相机，终点在远平面上）
```
// FS
float depthInBuffer = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
float linear01Depth = Linear01Depth(depthInBuffer);
float3 worldPos = _WorldSpaceCameraPos + i.farPlaneVector * linear01Depth;
```

## 3.NDC逆变换法
当渲染场景中的物体时，上面这种做法就不适用了，我们还可以使用NDC坐标来求出这条射线。
![image](https://user-images.githubusercontent.com/29577919/169684509-45933ceb-5175-404c-a24c-116d78fcd260.png)

首先计算出NDC坐标，ComputeScreenPos会处理不同平台UV的差异：
```
// VS
float4 screenPos = ComputeScreenPos(o.vertex);
float4 ndcPos = (o.screenPos / o.screenPos.w) * 2 - 1;
```
然后计算出Clip空间中，这条射线落在远平面的坐标：
```
// VS
float far = _ProjectionParams.z;
float4 clipPos = float4(ndcPos.x * far, ndcPos.y * far, far, far);
```
然后乘以VP矩阵的逆矩阵，计算出View空间中，这条射线落在远平面的坐标G：
```
// VS
float3 viewPos = mul(unity_CameraInvProjection, clipPos).xyz;
所以CG向量就是viewPos（起点在原点）：
```
```
// VS
float3 farPlaneVectorView = viewPos;
```
再把这个向量转换到世界空间：

```
// VS
o.farPlaneVector = mul(UNITY_MATRIX_I_V, float4(farPlaneVectorView, 0)).xyz;
```
剩下的处理就跟上面的方法一样了。
```
// FS
float depthInBuffer = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
float linear01Depth = Linear01Depth(depthInBuffer);
float3 worldPos = _WorldSpaceCameraPos + i.farPlaneVector * linear01Depth;
```
## 4.直接相减法
首先直接计算世界空间中Camera到顶点的向量：
```
float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
float3 worldSpaceViewVector = worldPos - _WorldSpaceCameraPos.xyz;
```
然后计算View空间下这个向量的深度值（View空间是右手坐标系，取正值）：
```
o.viewSpaceDepth = -mul(UNITY_MATRIX_V, float4(o.worldSpaceViewVector, 0.0)).z;
```
然后用当前顶点的深度和深度纹理中得到的深度的比例关系，求出向量：
```
float depthInBuffer = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
float linearDepth = LinearDepth(depthInBuffer);

float3 ray = i.worldSpaceViewVector * (linearDepth / i.viewSpaceDepth );
float3 worldPos = _WorldSpaceCameraPos + ray;
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
