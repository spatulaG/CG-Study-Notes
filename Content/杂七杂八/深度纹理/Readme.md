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
