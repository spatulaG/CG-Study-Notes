# 颜色处理

<img src="media/cover.jpeg" width="100" height="100">

## 视网膜包含两种细胞
- 视锥细胞： 对R(600nm)G(550)B(450)三个颜色敏感， 也是SML波长
- 视杆细胞： 对黑白敏感

可见光：400-750nm

CIE的等色匹配实验为了找到primary（三元光）和复色的配比得到了这张图：

<img src="https://upload.wikimedia.org/wikipedia/commons/3/36/CIE1931_RGBCMF.png" width="400" >

确定了RGB分别是 700，546.1和435.8//RGB
但450-550范围内的长光需要负值的红光才能匹配到，但是光只能加不能减。

### color matching functions

消灭R负值， 于是提出CIE XYZ， 三个想象出来的光。

<img src="media/01.png" width="400" >

R轴有负数

<img src="media/02.png" width="400" >

找到XYZ

<img src="media/03.png" width="400" >

变换成垂直坐标系。

但因为我们只想讨论颜色不想讨论亮度，所以把这个三维空间投影到了X+Y+Z=1的平面
成为CIE 1931 xyz color space舌形图
或者把z也丢掉只用XY表示色度

色域：gamut

# 颜色空间

颜色空间需要具备的三个条件：三原色，白点，传递函数
三原色可能在不同颜色空间对应不同的XY，白点是rgb是1时的XY位置（sRGB颜色空间的白点D65的位置如下（xy坐标为(0.31271, 0.32902)））
白点定义了这个RGB颜色空间中纯白色 (1, 1, 1)在色度图上的位置

利用三原色在色度图里的xy坐标可以它们在XYZ坐标空间中的向量方向，为了方便计算我们可以只考虑它们的单位向量方向记为(Rx, Ry, Rz)、(Gx, Gy, Gz)和(Bx, By, Bz)。通过缩放并叠加这三个单位向量方向我们可以得到这个颜色空间的任意一点的空间位置，即P = (Rx, Ry, Rz) * r + (Gx, Gy, Gz) * g + (Bx, By, Bz) * b，其中r、g、b分别表示三个向量的缩放大小。

那么，我们可以找到一组(r, g, b)系数使得P = 白点位置。也就是说，通过这个白点我们可以定义三原色向量的相对长度关系

白点实际上是RGB是1时在X+Y+Z=1平面上的投影，我们可以等比例放缩rgb，来得到白点所在的白色射线。
其中，当Y=1时， 亮度为最大是色彩的上限。

## 白点的推算思路：

首先x = X/X+Y+Z, y = Y/X+Y+Z, z = Z/X+Y+Z. x+y+z = 1.

- xyz:色坐标（投影到X + Y + Z = 1的平面上。这个二维色度空间（chromaticity color space）就被称为CIE 1931 xyz color space）
- XYZ：三刺激值

给出RY，GY，BY就可以算出RGB三刺激值

<img src="media/04.png" width="400" >

而白色坐标就是
- WX = RX+GX+BX
- WY...
- WZ...

然后给出白色的色坐标和亮度值也可以反推RGB三原色亮度。

- 然而我之前一直以为白点就是这么算出来的后来发现怎么算怎么不对， 原来我们不取1：1：1。

• 等量的三色
```
IEE(λ) = 1
(xEE, yEE, zEE) = (1/3, 1/3, 1/3)
```

```
• D65 光照 (PAL):
I65(λ) = Natural Sun Light
(x65, y65, z65) = (0.3127, 0.3290, 0.3583)
```

```
• C 光照 (NTSC):
Ic(λ) = not defined
(xc, yc, zc) = (0.310, 0.316, 0.374)
```

```
// 我们需要求解新的RGB颜色空间（sRGB）的三原色在XYZ颜色空间的索引值：
Rxyz = (Rx, Ry, Rz)
Gxyz = (Gx, Gy, Gz)
Bxyz = (Bx, By, Bz)

// 通过色度图中的三原色和白点坐标，我们可知：
Rxyz = (0.64, 0.33, 0.03) * r
Gxyz = (0.30, 0.60, 0.10) * g
Bxyz = (0.15, 0.06, 0.79) * b

// 白点在XYZ颜色空间的索引值为
Wxyz = (Wx, 1, Wz)
     = (0.31271, 0.32902, 0.35827) * w
     = |Rx Gx Bx| |1|
       |Ry Gy By| |1|
       |Rz Gz Bz| |1|

// 那么有：
w = 3.03933
Wxyz = (0.95043, 1, 1.08890)

// 结合两者有：
0.64r + 0.30g + 0.15b = 0.95043
0.33r + 0.60g + 0.06b = 1
0.03r + 0.10g + 0.79b = 1.08890

// 求解可得：
r = 0.644463125
g = 1.191920333
b = 1.202916667

// 那么sRGB颜色空间的三原色在XYZ颜色空间的索引值为：
Rxyz = (0.4124564, 0.2126729, 0.0193339)
Gxyz = (0.3575761, 0.7151522, 0.1191920)
Bxyz = (0.1804375, 0.0721750, 0.9503041)
```

## 引用：

[具体实验参考](https://medium.com/hipster-color-science/a-beginners-guide-to-colorimetry-401f1830b65a)

[COD ppt](https://research.activision.com/publications/archives/hdr-in-call-of-duty)

[HDR for CG artists](https://chrisbrejon.com/cg-cinematography/)

[wiki](https://en.wikipedia.org/wiki/RGB_color_spaces)
