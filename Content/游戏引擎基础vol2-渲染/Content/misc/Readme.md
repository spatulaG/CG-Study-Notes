# 补充内容

## LUT

一种转换模型。传统的1D LUT变动某个颜色输入值只会影响到该颜色的输出值，RGB的数据之间是互相独立的。
1D LUTs只能控制gamma值、RGB平衡（灰阶）和白点（white point）。

为了克服1D LUT的限制，我们可以对其应用一个3×3的矩阵来控制颜色饱和度和亮度，在完整的色彩空间中进行线性缩放。但更好的办法是3D LUT。

![](LUT3D.jpg)

```
//check the point in pos
bool checkPointInRect(uint2 point,uint2 origin, uint2 rect){
    return point.x >= origin.x &&
    point.y >= origin.y &&
    point.x <= (origin.x + rect.x) &&
    point.y <= (origin.y + rect.y);
}
kernel void image_filiter(constant ImageSaturationParams *params [[buffer(0)]],
                          texture2d<half, access::sample> sourceTexture [[texture(0)]],
                          texture2d<half, access::write> targetTexture [[texture(1)]],
                          texture2d<half, access::sample> lutTexture [[texture(2)]],
                          sampler samp [[sampler(0)]],
                          uint2 gridPos [[thread_position_in_grid]]){

    
    float2 sourceCoord = float2(gridPos);
    half4 color = sourceTexture.sample(samp,sourceCoord);
    
    
    float blueColor = color.b * 63.0;
    
    int2 quad1;
    quad1.y = floor(floor(blueColor) / 8.0);
    quad1.x = floor(blueColor) - (quad1.y * 8.0);
    
    int2 quad2;

    quad2.y = floor(ceil(blueColor) / 8.0);
    quad2.x = ceil(blueColor) - (quad2.y * 8.0);
    
    half2 texPos1;
    texPos1.x = (quad1.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * color.r);
    texPos1.y = (quad1.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * color.g);
    
    half2 texPos2;
    texPos2.x = (quad2.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * color.r);
    texPos2.y = (quad2.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * color.g);
    
    
    half4 newColor1 = lutTexture.sample(samp,float2(texPos1.x * 512 ,texPos2.y * 512));
    half4 newColor2 = lutTexture.sample(samp,float2(texPos2.x * 512,texPos2.y * 512 ));
  
    half4 newColor = mix(newColor1, newColor2, half(fract(blueColor)));
    
    
    half4 finalColor = mix(color, half4(newColor.rgb, color.w), half(params->saturation));
    
 
    uint2 destCoords = gridPos + params->clipOrigin;
    
    
    uint2 transformCoords = destCoords;
    
    //transform coords for y
    if (params->changeCoord){
        transformCoords = uint2(destCoords.x, sourceTexture.get_height() - destCoords.y);
    }
    //transform color for r&b
    half4 realColor = finalColor;
    if (params->changeColor){
        realColor = half4(finalColor.bgra);
    }
    
    if(checkPointInRect(transformCoords,params->clipOrigin,params->clipSize))
    {
        targetTexture.write(realColor, transformCoords);
        
    }else{
        
        targetTexture.write(color,transformCoords);
    }
}
```

[来源](https://programmer.ink/think/metal-series-of-tutorials-2-metal-implementation-of-lut-filters.html)
举例说明的话就是：
First, we determine which square b = 0.2 * 63 = 12.6, that is (4, 1) which one to use.
r = 0.4 * 63 = 25.6, g = 0.6 * 63 = 37.8 to convert to macrocoordinates (4 * 64 + 25.6, 1 * 64 + 37.8)
The first three steps are all floating-point numbers, but the pixels of our filters are fixed and there are no decimal numbers.
For r,g finally converts the arrival coordinates to normalized coordinates, ((4*64+25.6)/512, (1*64+37.8)/512), and extracts the exact color values by sampler interpolation.
For b, we can take the next box (5,1) and mix the two colors to get the final color.
