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
···
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
···


采样得到的深度值范围是0~1，NDC空间下深度值范围是-1~1，所以有：

[公式]

NDC坐标是裁剪空间坐标经过齐次除法得到的，所以有：

[公式]

定义Far为远平面距离，Near为近平面距离，根据投影变换，可以得到Zclip和Zview的计算公式：

[公式]

[公式]

所以反过来可以得出Zview和Zclip的关系：

[公式]

代入Zclip和Zndc，可以求出：

[公式]
